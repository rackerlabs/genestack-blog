---
date: 2026-04-08
title: Fixing Duplicate Kube-OVN IP Allocations Caused by Stale IP CRDs
authors:
  - rackerlabs
description: >
  Learn how to identify duplicate Kube-OVN pod IP allocations, confirm when a stale IP CRD is blocking a new pod, and safely remove the orphaned record.
categories:
  - ovn
  - openstack
  - kubernetes
  - networking
---

# Fixing Duplicate Kube-OVN IP Allocations Caused by Stale IP CRDs

When a pod fails to join the network in a Kube-OVN-backed cluster, the first symptom often looks like a generic CNI problem. In one Genestack-operated IAD sandbox case, the actual cause was a duplicate IP allocation: a new pod was assigned an address that was still recorded against an older, non-running pod in the same subnet.

<!-- more -->

The result was a failed `ADD` request from the CNI server, repeated gateway ping failures, and a pod that could not complete network setup. This post walks through how to confirm the overlap, inspect the subnet state, find duplicate IP records across the cluster, and remove the stale Kube-OVN IP CRD that is blocking the replacement workload.

This is an operator procedure for a Genestack-managed environment with direct Kubernetes administrative access. It is not a normal tenant-side OpenStack networking workflow; it is a control-plane and platform-operations recovery path.

## The symptom

The affected pod was scheduled on node `1329327-mgmt-prod.iad3.ohthree.com` as `nova-service-cleaner-29363820-vj5z8`. Its pod annotations showed that Kube-OVN had allocated `10.236.3.166/14` on subnet `ovn-default` with gateway `10.236.0.1`.

``` text
Annotations:
  ovn.kubernetes.io/allocated: true
  ovn.kubernetes.io/cidr: 10.236.0.0/14
  ovn.kubernetes.io/gateway: 10.236.0.1
  ovn.kubernetes.io/ip_address: 10.236.3.166
  ovn.kubernetes.io/logical_router: ovn-cluster
  ovn.kubernetes.io/logical_switch: ovn-default
  ovn.kubernetes.io/mac_address: 62:09:0c:f0:a4:dc
  ovn.kubernetes.io/pod_nic_type: veth-pair
  ovn.kubernetes.io/routed: true
```

At the same time, the `kube-ovn-cni` logs showed that the interface was created, but the pod never received packets back from the gateway:

``` text
cni-server W1030 17:07:35.628122 ovs.go:35] 10.236.3.166 network not ready after 198 ping to gateway 10.236.0.1
cni-server W1030 17:07:37.636453 ovs.go:71] 10.236.3.166 network not ready after 200 ping to gateway 10.236.0.1
cni-server E1030 17:07:37.636480 ovs_linux.go:595] no packets received from gateway 10.236.0.1
cni-server E1030 17:07:37.636523 ovs_linux.go:202] no packets received from gateway 10.236.0.1
cni-server I1030 17:07:37.869295 ovs_linux.go:349] rollback ovs port success <redacted>
cni-server I1030 17:07:37.907854 ovs_linux.go:1982] rollback veth success <redacted>
cni-server E1030 17:07:37.907891 handler.go:320] configure nic eth0 for pod nova-service-cleaner-29363820-vj5z8/openstack failed: no packets received from gateway 10.236.0.1
cni-server I1030 17:07:37.907955 server.go:74] [2025-10-30T17:07:37Z] Outgoing response POST /api/v1/add with 500 status code in 200477ms
```

That combination is a strong signal that the IP assignment is not clean, even if the pod annotation looks valid.

## Confirm the overlapping IP allocation

The fastest way to validate the suspicion is to search the Kube-OVN IP CRDs for the address the failing pod received:

``` shell
kubectl get ip -A | grep 10.236.3.166
```

In this case, the same IP appeared twice:

``` text
loki-read-5884dbbfff-rm49q.grafana            10.236.3.166  42:8f:ee:74:12:b8  1329322-mgmt-prod.iad3.ohthree.com  ovn-default
nova-service-cleaner-29363820-vj5z8.openstack 10.236.3.166  62:09:0c:f0:a4:dc  1329327-mgmt-prod.iad3.ohthree.com  ovn-default
```

This is the critical clue. A stale IP record for `loki-read-5884dbbfff-rm49q.grafana` was still present, so Kube-OVN ended up with a conflicting assignment recorded for a different pod.

## Inspect the subnet state

It is also useful to inspect the subnet object directly:

``` shell
kubectl get subnet ovn-default -o yaml
```

From the subnet output, the important details were:

``` text
spec:
  cidrBlock: 10.236.0.0/14
  default: true
  gateway: 10.236.0.1
  gatewayType: distributed
  natOutgoing: true
  protocol: IPv4
status:
  v4availableIPs: 260814
  v4usingIPs: 1327
```

The full `v4availableIPrange` and `v4usingIPrange` lists were very large, but the key takeaway was that the subnet itself still looked broadly healthy. The failure was not caused by total address exhaustion. It was caused by a bad record inside the allocated IP set.

## Find duplicate IPs across the cluster

If you suspect the issue may not be isolated to a single pod, you can scan every Kube-OVN IP record and report duplicates. The following helper script lists duplicate IPs and also checks whether the referenced pod still exists.

``` shell
#!/bin/bash

echo "--- Fetching all Kube-OVN IP records (ips.kubeovn.io) across all namespaces..."
echo "Note: This command relies on the Kube-OVN 'IP' Custom Resource Definition (CRD)."

IP_DATA=$(kubectl get ip -A -o custom-columns=IPADDRESS:.spec.ipAddress,NAMESPACE:.spec.namespace,PODNAME:.spec.podName --no-headers 2>/dev/null)

if [ -z "$IP_DATA" ]; then
    echo "ERROR: No Kube-OVN 'IP' records found. Ensure Kube-OVN is running correctly." >&2
    exit 1
fi

ALL_IPS=$(echo "$IP_DATA" | awk '{print $1}')
DUPLICATE_IPS=$(echo "$ALL_IPS" | sort | uniq -c | awk '$1 > 1 {print $2}')

check_pod_status() {
    local ns="$1"
    local pod="$2"
    if kubectl get pod "$pod" -n "$ns" --no-headers &>/dev/null; then
        echo "Active"
    else
        echo "Missing"
    fi
}
export -f check_pod_status

if [ -z "$DUPLICATE_IPS" ]; then
    echo ""
    echo "*** Success: No duplicate Kube-OVN IP addresses found across the cluster."
else
    echo ""
    echo "*** Duplicate IP Addresses Found! ATTENTION"
    echo "-------------------------------------"

    echo "$DUPLICATE_IPS" | while read DUP_IP; do
        echo "IP: ${DUP_IP} is used by:"
        echo "$IP_DATA" | grep "^${DUP_IP}\s" | while read -r IP NS POD; do
            POD_STATUS=$(check_pod_status "$NS" "$POD")
            IP_CRD_NAME="${POD}.${NS}"

            if [[ "$POD_STATUS" == "Active" ]]; then
                echo -e "\t- namespace: ${NS} podname: ${POD} | Status: ${POD_STATUS}"
            else
                echo -e "\t- *** namespace: ${NS} podname: ${POD} | Status: ${POD_STATUS} (Stale CRD?)"
                echo -e "\t= *** Resolution: kubectl delete ip ${IP_CRD_NAME}"
            fi
        done
    done
    echo "-------------------------------------"
    echo "Action Required: Investigate the pods and the kube-ovn-controller logs."
    echo "Duplicate IPs and 'Missing' Pods often indicate stale IP records (CRDs) that need cleanup."
fi
```

This approach is especially useful when duplicate assignments only surface after a pod restart or a controller cleanup failure.

## Remove the stale IP CRD

Once you confirm that one of the duplicate IP records belongs to a pod that no longer exists, remove the stale `ip` object by name. In this case, the non-running pod record was the `grafana` entry:

``` shell
kubectl delete ip loki-read-5884dbbfff-rm49q.grafana
```

That frees the duplicate allocation and allows Kube-OVN to stop colliding with the stale record.

## Verify the fix

After deleting the orphaned IP CRD, re-check the duplicate assignment:

``` shell
kubectl get ip -A | grep 10.236.3.166
```

You should see only the active pod holding the address. At that point, the replacement pod can be recreated or retried and should be able to attach its interface cleanly without the repeated `no packets received from gateway` failures.

## Summary

In a Genestack-managed OpenStack environment, when a control-plane or supporting-cluster pod receives a valid-looking Kube-OVN annotation but still cannot reach its gateway, check for duplicate IP CRDs before assuming a deeper OVS or routing issue. In this case, the root cause was a stale Kube-OVN `ip` object left behind for a non-running pod, and deleting that orphaned record immediately resolved the overlap.
