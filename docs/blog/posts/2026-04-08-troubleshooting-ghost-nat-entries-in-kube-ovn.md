---
date: 2026-04-08
title: Troubleshooting Ghost NAT Entries in Kube-OVN When Floating IPs Go Intermittent
authors:
  - chrisbreu
description: >
  Learn how to identify and remove stale OVN NAT entries that can cause intermittent OpenStack Neutron Floating IP connectivity while using Kube-OVN tooling to inspect OVN.
categories:
  - ovn
  - openstack
  - networking
---

# Troubleshooting Ghost NAT Entries in Kube-OVN When Floating IPs Go Intermittent

In a large-scale OpenStack environment, especially one leveraging Genestack, networking is the lifeblood of the platform. One of the more frustrating issues operators can face is the intermittent Floating IP: connectivity works for a while, then drops unexpectedly, or only succeeds from certain source networks.

<!-- more -->

In this deployment model, OpenStack Neutron is managing Floating IP lifecycle while OVN provides the underlying logical networking. The `kubectl ko nbctl` command is simply an operator-side way to inspect OVN through the Kube-OVN plugin in a Genestack-managed environment; it is not a normal tenant workflow. In many cases, the issue is not a physical link problem or a firewall rule. The root cause can instead be a stale NAT entry in the OVN Northbound database (NBDB) that conflicts with a legitimate Floating IP mapping.

## The Symptom: Intermittent Connectivity

The classic sign of a duplicate or stale NAT entry is inconsistency. OVN may have more than one `dnat_and_snat` mapping for the same external IP, with different logical destinations or metadata. In that state, stale NBDB NAT configuration can continue to be realized in OVN until it is removed.

## Step 1: Hunt for duplicate NAT entries

Start by inspecting the OVN Northbound database. Using the `kubectl ko` Kube-OVN plugin, query `nbctl` for all `dnat_and_snat` entries and look for duplicate external IPs.

``` shell
kubectl ko nbctl find NAT type=dnat_and_snat | grep external_ip | sort | uniq -d
```

If this command returns an IP address such as `50.xx.xx.20`, you likely have a conflict.

## Step 2: Inspect the conflicting entries

Once you identify a duplicate IP, pull the full details for those entries and compare the logical IPs, any `logical_port` values, and the Neutron metadata attached to each mapping.

``` shell
kubectl ko nbctl find NAT type=dnat_and_snat external_ip="50.xx.xx.20"
```

The output may look similar to the following:

``` text
_uuid               : 896fc1d4-cb11-4df2-b8a0-cab029664595
external_ids        : {"neutron:fip_id"="84b12c2f-...", "neutron:router_name"="neutron-bbe161cd-..."}
external_ip         : "50.xx.xx.20"
logical_ip          : "10.0.4.10"
logical_port        : "4b6d7f3b-..."

--

_uuid               : f81f3052-bcdd-4a1b-879b-c37127fe18ce
external_ids        : {"neutron:fip_id"="947a90cf-...", "neutron:router_name"="neutron-bd8317a3-..."}
external_ip         : "50.xx.xx.20"
logical_ip          : "10.0.18.99"
```

The key signal is that the same `external_ip` appears in more than one NAT record, each tied to a different logical destination and different OpenStack metadata.

## Step 3: Validate the entries against Neutron

In an OpenStack-managed deployment, Neutron is the source of truth for Floating IP lifecycle. The next step is to verify which Floating IP record still exists.

``` shell
openstack floating ip show 947a90cf-51c2-4a3c-b21a-247a8d4bc7e1
```

If the command returns an error such as `No FloatingIP found`, you have identified the stale OVN entry. In other words, Neutron has already deleted the Floating IP, but OVN NBDB is still holding the old NAT record.

## Step 4: Identify the logical router that owns the stale NAT entry

Before deleting anything, confirm which logical router owns the stale NAT rule. In many OpenStack environments, the router name is available in the NAT entry's `external_ids`, but you can also confirm it directly in OVN by checking which logical router references the stale NAT UUID.

``` shell
kubectl ko nbctl find logical_router nat{>=}f81f3052-bcdd-4a1b-879b-c37127fe18ce
```

You can also inspect the candidate router directly if you derived its name from the NAT metadata:

``` shell
kubectl ko nbctl find logical_router name=neutron-bd8317a3-...
```

Once you confirm the owning logical router, you are ready to remove the stale entry.

## Step 5: Remove the stale OVN NAT entry

To restore stable connectivity, manually prune the stale record from the OVN Northbound database. Use the confirmed logical router name together with the external IP from the stale record:

``` shell
# Format: kubectl ko nbctl lr-nat-del <router_name> <type> <external_ip>
kubectl ko nbctl lr-nat-del neutron-bd8317a3-07a9-4764-902b-abcc57fa640f dnat_and_snat 50.xx.xx.20
```

This removes the bad NAT mapping from the affected logical router.

## Step 6: Verify the cleanup

After deleting the stale entry, verify that only the valid mapping remains:

``` shell
kubectl ko nbctl find NAT type=dnat_and_snat | grep "50.xx.xx.20"
```

The output should now show only a single clean NAT record for that external IP.

## Summary

In a dynamic cloud environment, race conditions or failed cleanup tasks can occasionally leave behind stale synchronization artifacts between Neutron and OVN. By treating Neutron as the source of truth and comparing it with OVN Northbound state, you can quickly identify stale Floating IP mappings, confirm the owning logical router, and remove the bad entry before it continues to cause intermittent traffic failures.

## Pro Tip

If this issue appears frequently, it is worth auditing `neutron-server` logs during Floating IP deletion events. That can help confirm whether the OVN mechanism driver is successfully communicating cleanup operations to OVSDB and whether stale NAT state is being left behind during cleanup.
