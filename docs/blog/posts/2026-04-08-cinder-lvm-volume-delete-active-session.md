---
date: 2026-04-08
title: Fixing Cinder LVM Volume Deletion Failures Caused by Active iSCSI Sessions
authors:
  - chrisbreu
description: >
  Learn how to identify and clear active tgt iSCSI sessions that can prevent OpenStack Cinder from deleting an LVM-backed volume.
categories:
  - openstack
  - cinder
  - storage
---

# Fixing Cinder LVM Volume Deletion Failures Caused by Active iSCSI Sessions

In OpenStack environments using the Cinder LVM backend with `tgt`, a volume deletion can fail even after the instance side of the workflow appears complete. One common cause is that the iSCSI session on the block storage node is still active, preventing `tgt` from removing the target.

<!-- more -->

This recovery flow is specifically for deployments that still expose Cinder volumes through the LVM plus `tgt` iSCSI path. It is a useful operator procedure for older or specialized backends, but it should not be read as the default storage pattern for every Genestack release.

When this happens, Cinder may leave the volume in an `error_deleting` state and log a target removal failure similar to `tgtadm: this target is still active`.

All identifiers and addresses in the examples below have been obfuscated.

## The Symptom

The failure usually shows up in the `cinder-volume` service logs when Cinder tries to tear down the exported target for the volume.

``` log
ERROR oslo_messaging.rpc.server Exception during message handling: cinder.exception.ISCSITargetRemoveFailed: Failed to remove iscsi target for volume <volume_uuid_redacted>.
ERROR oslo_messaging.rpc.server Traceback (most recent call last):
ERROR oslo_messaging.rpc.server   File "/opt/cinder/lib/python3.12/site-packages/cinder/volume/targets/tgt.py", line 248, in remove_iscsi_target
ERROR oslo_messaging.rpc.server     cinder.privsep.targets.tgt.tgtadmin_delete(iqn, force=True)
ERROR oslo_messaging.rpc.server   File "/opt/cinder/lib/python3.12/site-packages/oslo_privsep/priv_context.py", line 271, in _wrap
ERROR oslo_messaging.rpc.server     return self.channel.remote_call(name, args, kwargs,
ERROR oslo_messaging.rpc.server   File "/opt/cinder/lib/python3.12/site-packages/oslo_privsep/daemon.py", line 214, in remote_call
ERROR oslo_messaging.rpc.server     raise exc_type(*result[2])
ERROR oslo_messaging.rpc.server oslo_concurrency.processutils.ProcessExecutionError: Unexpected error while running command.
ERROR oslo_messaging.rpc.server Command: tgt-admin --delete iqn.2010-10.org.openstack:<target_uuid_redacted> -f
ERROR oslo_messaging.rpc.server Exit code: 22
ERROR oslo_messaging.rpc.server Stdout: 'Command: tgtadm -C 0 --mode target --op delete --tid=7 exited with code: 22.'
ERROR oslo_messaging.rpc.server Stderr: 'tgtadm: this target is still active'
```

This is the key indicator: the target cleanup is failing because `tgt` still sees an attached initiator session.

## Step 1: Inspect active `tgt` sessions

On the block storage node, inspect the exported targets and look for the volume target that is stuck active.

``` shell
tgtadm --mode target --op show
```

Example output:

``` text
Target 7: iqn.2010-10.org.openstack:<target_uuid_redacted>
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
        I_T nexus: 298
            Initiator: 01:<initiator_id_redacted> alias: <compute_name_redacted>
            Connection: 0
                IP Address: 172.26.xx.xx
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00070000
            SCSI SN: <controller_sn_redacted>
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags:
        LUN: 1
            Type: disk
            SCSI ID: <volume_uuid_redacted>
            SCSI SN: <volume_uuid_redacted>
            Size: 171799 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/cinder-volumes-1/<volume_uuid_redacted>
            Backing store flags:
```

The important details are the target ID, session ID, and connection ID. In this example those are `7`, `298`, and `0`.

## Step 2: Force logout of the active session

If the attached compute no longer needs the device and the session is truly stale, force the connection to log out.

``` shell
tgtadm --lld iscsi --op delete --mode conn --tid 7 --sid 298 --cid 0 --force
```

This forcibly closes the active connection that is blocking target deletion. However, this is only effective if the initiator is no longer actively using or retrying the session. If the compute host still has the volume attached, the initiator may immediately log back in with a new session ID and connection ID, and the target delete can fail again.

## Step 3: Remove the target if needed

After closing the session, verify that the initiator does not reconnect. If the session is gone and the target still needs to be cleaned up, delete the target directly.

``` shell
tgtadm -C 0 --mode target --op delete --tid=7 --force
```

If `tgtadm` still reports that the target is active, re-check `tgtadm --mode target --op show` for a newly established session before retrying.

## Step 4: Recover a volume stuck in `error_deleting`

If the failed cleanup already pushed the volume into `deleting` or `error_deleting`, retry the delete using the force option first.

``` shell
openstack volume delete --force <volume_uuid_redacted>
openstack volume show <volume_uuid_redacted>
```

In many cases, this is the correct follow-up once the stale iSCSI session has been cleared.

If the volume state itself is stuck and an administrator needs to override it manually, reset the state with caution and then retry the delete:

``` shell
openstack volume set --state error <volume_uuid_redacted>
openstack volume delete <volume_uuid_redacted>
openstack volume show <volume_uuid_redacted>
```

The `volume set --state` command is an administrative recovery action rather than the normal first step, because it overrides the recorded Cinder state for the volume.

## Why This Happens

In most cases, the compute-side disconnect did not fully propagate before Cinder attempted to delete the export. The LVM volume itself may be ready for removal, but `tgt` still tracks an initiator session against the target, so `tgt-admin --delete` fails and Cinder surfaces the backend exception.

## Summary

If your OpenStack or Genestack environment uses the Cinder LVM backend with `tgt`, and `cinder-volume` reports that the iSCSI target is still active, check the block node for lingering `tgt` sessions. Identifying the active target with `tgtadm --mode target --op show`, ensuring the compute host is no longer reconnecting, and then retrying the delete with `openstack volume delete --force` is usually enough to resolve the issue cleanly.
