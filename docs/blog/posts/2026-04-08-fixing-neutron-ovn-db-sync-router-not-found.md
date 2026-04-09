---
date: 2026-04-08
title: Fixing neutron-ovn-db-sync-util RouterNotFound Errors During OVN ACL Reconciliation
authors:
  - chrisbreu
description: >
  Learn how to recover when neutron-ovn-db-sync-util fails with RouterNotFound by identifying and deleting a stale Neutron port that references a missing router.
categories:
  - openstack
  - neutron
  - ovn
  - networking
---

# Fixing neutron-ovn-db-sync-util RouterNotFound Errors During OVN ACL Reconciliation

When Neutron and OVN drift out of sync, one of the standard recovery tools is `neutron-ovn-db-sync-util`. In some environments, though, the sync itself can fail before it repairs anything, especially if Neutron still contains stale objects that reference routers that no longer exist.

<!-- more -->

One example is a `RouterNotFound` exception during ACL reconciliation. In that case, the database sync utility is usually telling you that it found a dangling dependency in Neutron that must be cleaned up before the OVN add-mode sync can complete.

All environment names, hostnames, request identifiers, router IDs, and port IDs in the examples below have been obfuscated.

## Summary

An OVN database reconciliation attempt failed with `neutron_lib.exceptions.l3.RouterNotFound` while running `neutron-ovn-db-sync-util` in `add` mode in an ML2/OVN-backed OpenStack environment. The failure itself, not a raw object-count comparison, was the signal that a stale Neutron dependency was blocking reconciliation.

The root cause was a stale Neutron port that still referenced a router UUID that no longer existed. Deleting the orphaned port removed the bad dependency and allowed the environment to move forward with synchronization.

## The Symptom

The most important symptom in this case was the sync utility itself failing before it could repair OVN state. In some incidents you may also notice other signs of Neutron and OVN drift, but raw Neutron security group rule counts and raw OVN ACL counts are not a reliable one-to-one health check and should be treated only as anecdotal clues.

To prepare the sync, copy the active Neutron configuration files and remove any existing `neutron_sync_mode` setting from the temporary copies:

``` shell
cp /etc/neutron/neutron.conf /etc/neutron/plugins/ml2/ml2_conf.ini /tmp/
sed -i 's/neutron_sync_mode.*//g' /tmp/neutron.conf /tmp/ml2_conf.ini
neutron-ovn-db-sync-util --ovn-neutron_sync_mode add --config-file /tmp/neutron.conf --config-file /tmp/ml2_conf.ini
```

In the affected environment, the utility failed immediately with the following error:

``` text
2025-12-05 15:01:21.913 108 CRITICAL neutron_ovn_db_sync_util [None req-<redacted> - - - - - -] Unhandled error: neutron_lib.exceptions.l3.RouterNotFound: Router <router_uuid_redacted> could not be found
```

That error is the important clue. The sync utility is walking Neutron state and encountered an object that still points to a router record that has already been removed.

## Step 1: Identify the stale dependency

The next step is to inspect the Neutron port that still references the missing router:

``` shell
openstack port list --device-id <router_uuid_redacted>
```

If this returns a port even though the router no longer exists, you have found the stale object blocking the sync.

In this case, the result was an orphaned port still attached to the deleted router UUID.

## Step 2: Delete the orphaned port

Once you confirm the router is gone and the port is genuinely stale, delete the orphaned port:

``` shell
openstack port delete <port_uuid_redacted>
```

This removes the bad Neutron-side reference that caused `neutron-ovn-db-sync-util` to fail with `RouterNotFound`.

## Step 3: Re-run the OVN add sync

After removing the stale port, rerun the sync using the same temporary configuration files:

``` shell
neutron-ovn-db-sync-util --ovn-neutron_sync_mode add --config-file /tmp/neutron.conf --config-file /tmp/ml2_conf.ini
```

With the orphaned dependency removed, the utility should be able to continue reconciling Neutron state into OVN.

## Why This Happens

This failure pattern usually means Neutron still contains a dependent object, often a port or router-related interface record, whose parent router has already been deleted. Under normal cleanup paths those references are removed automatically, but if a workflow is interrupted or partially fails, the stale dependency can survive long enough to break later reconciliation.

The sync utility is not creating the inconsistency here. It is surfacing one that already exists.

## Summary

If `neutron-ovn-db-sync-util` fails with `RouterNotFound`, do not focus only on the missing router itself. Check for stale Neutron ports or related resources that still reference that router UUID. Deleting the orphaned port and then rerunning the OVN add sync is often the cleanest fix.
