---
date: 2026-04-08
title: Fixing Nova Reboot Failures When QEMU Cannot Access /dev/kvm
authors:
  - chrisbreu
description: >
  Learn how to identify and recover from libvirt and QEMU KVM permission errors caused by a /dev/kvm ownership or GID mapping mismatch between the host and the libvirt pod.
categories:
  - openstack
  - nova
  - libvirt
  - kubernetes
---

# Fixing Nova Reboot Failures When QEMU Cannot Access /dev/kvm

In containerized OpenStack compute environments, a hard reboot or instance start can fail even when the hypervisor node itself looks healthy. One failure mode is a permissions mismatch on `/dev/kvm`, where the device inside the libvirt pod is mapped with ownership or permissions that do not line up with the host device.

<!-- more -->

When that happens, `qemu-system-x86_64` cannot initialize KVM acceleration, libvirt loses the monitor unexpectedly, and Nova surfaces the failure during the guest launch or reboot path.

All tenant, request, host, and instance identifiers in the examples below have been obfuscated.

## Summary

An instance reboot was failing because `qemu-system-x86_64` inside the libvirt pod could not access `/dev/kvm`. The immediate symptom was `failed to initialize kvm: Permission denied`, followed by a Nova/libvirt exception when the guest launch failed.

The underlying issue in this Genestack deployment was a `/dev/kvm` ownership mismatch between the host and the libvirt pod. The important detail was the effective numeric ownership and permissions on the device, not just the textual group label shown by `ls`:

- Inside the pod, `/dev/kvm` was exposed with a different effective group mapping than the host device
- On the node, `/dev/kvm` ownership had drifted away from what the libvirt daemonset expected during guest startup

The resolution was to perform a rolling restart of the libvirt daemonset so the Genestack libvirt init logic could re-apply the expected device ownership and permissions on the affected hosts.

## Problem Statement

The first visible symptom was in the Nova and libvirt error path during a reboot operation:

``` log
ERROR nova.virt.libvirt.guest libvirt.libvirtError: internal error: qemu unexpectedly closed the monitor: Could not access KVM kernel module: Permission denied
ERROR nova.virt.libvirt.guest qemu-system-x86_64: -accel kvm: failed to initialize kvm: Permission denied
```

The corresponding sanitized traceback looked like this:

``` log
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server [req-<redacted> req-<redacted> <project_id_redacted> <user_id_redacted> - - <instance_id_redacted> <instance_id_redacted>] Exception during message handling: libvirt.libvirtError: internal error: qemu unexpectedly closed the monitor: Could not access KVM kernel module: Permission denied
2025-11-03T18:00:02.562872Z qemu-system-x86_64: -accel kvm: failed to initialize kvm: Permission denied
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server Traceback (most recent call last):
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server   File "/var/lib/openstack/lib/python3.12/site-packages/oslo_messaging/rpc/server.py", line 165, in _process_incoming
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server   File "/var/lib/openstack/lib/python3.12/site-packages/nova/compute/manager.py", line 4266, in reboot_instance
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server   File "/var/lib/openstack/lib/python3.12/site-packages/nova/virt/libvirt/driver.py", line 4130, in _hard_reboot
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server   File "/var/lib/openstack/lib/python3.12/site-packages/nova/virt/libvirt/driver.py", line 8056, in _create_guest_with_network
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server   File "/var/lib/openstack/lib/python3.12/site-packages/nova/virt/libvirt/driver.py", line 7973, in _create_guest
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server   File "/var/lib/openstack/lib/python3.12/site-packages/nova/virt/libvirt/guest.py", line 167, in launch
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server   File "/var/lib/openstack/lib/python3.12/site-packages/libvirt.py", line 1409, in createWithFlags
2025-11-03 18:00:03.552 ERROR oslo_messaging.rpc.server libvirt.libvirtError: internal error: qemu unexpectedly closed the monitor: Could not access KVM kernel module: Permission denied
```

That error chain makes it clear that Nova successfully reached the point where libvirt attempted to launch the guest, but QEMU could not open KVM acceleration on the compute host.

The next step was to compare the device ownership from inside the libvirt pod with the device ownership on the node. To avoid being misled by group names alone, compare the numeric UID and GID:

Pod view:

``` shell
root@<libvirt_pod_redacted>:/dev# ls -ln /dev/kvm
crw-rw---- 1 0 424 10, 232 Nov  3 19:40 /dev/kvm
```

Node view:

``` shell
root@<compute_node_redacted>:/dev# ls -ln /dev/kvm
crw-rw---- 1 0 108 10, 232 Nov  3 19:40 /dev/kvm
```

In this case the numeric group mapping did not line up between the pod and the host. That mismatch explained the behavior: QEMU was launched from the libvirt pod, but the device permissions reflected a different host-side mapping than the daemonset expected, leaving QEMU unable to access `/dev/kvm` during guest startup.

## Resolution

The fix was a rolling restart of the libvirt daemonset. Restarting the pods caused the Genestack libvirt init scripts to run again and restore the expected `/dev/kvm` ownership and permissions across the affected hosts.

In practice, the recovery flow was:

1. Confirm the failure in Nova and libvirt logs with `failed to initialize kvm: Permission denied`.
2. Compare the numeric ownership of `/dev/kvm` inside the libvirt pod and on the node with `ls -ln` or `stat`.
3. Perform a rolling restart of the libvirt daemonset.
4. Verify that the restarted pods re-applied the expected ownership and permissions and that instance reboot or launch operations succeeded again.

If multiple compute nodes are affected, a controlled rolling restart is the safest approach because it re-runs the same initialization logic consistently without requiring ad hoc manual permission changes on individual hosts.

## Why This Happens

In a Kubernetes-managed libvirt deployment, device permissions and group mappings are often prepared during pod startup. If the host-side device ownership drifts, or if the expected numeric GID mapping is not correctly reflected at runtime, the containerized libvirt process may still see `/dev/kvm` but be unable to use it for hardware acceleration.

That leads to a somewhat misleading failure pattern: the VM launch path proceeds normally through Nova until QEMU actually tries to initialize KVM, at which point the monitor exits and libvirt reports an internal error.

## Deep Dive

The following upstream changes provide additional implementation context around the libvirt and image handling involved in this fix:

- [genestack-images pull request #129](https://github.com/rackerlabs/genestack-images/pull/129)
- [genestack pull request #1285](https://github.com/rackerlabs/genestack/pull/1285)
