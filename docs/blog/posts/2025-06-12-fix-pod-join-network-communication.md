---
date: 2025-06-12
title: Repairing pod communication problems with the _join_ network
authors:
  - awfabian-rs
description: >
  This document describes
categories:
  - ovn
---

# Background

While converting a production region to _Kube-OVN_ managed by Helm chart from
its original _kubespray_ installation as detailed in the _Genestack_ doc
[docs/k8s-cni-kube-ovn-helm-conversion.md](https://github.com/rackerlabs/genestack/blob/main/docs/k8s-cni-kube-ovn-helm-conversion.md),
we encountered an issue with some of the `kube-ovn-cni` pods when they restarted
with an inability to communicate with the _join_ network, and crash looping or
pending pod states. We found that a reboot of the pod's node generally solved
this problem, but also discovered a method that could repair connectivity with
the _join_ network without rebooting the node.

We saw errors like this in the output of `kubectl logs` for the pod:

```
│ cni-server W0610 19:39:31.974287  476942 ovs.go:35] 100.64.0.118 network not ready after 195 ping to gateway 100.64.0.1                                                                                                                                                         │
│ cni-server W0610 19:39:34.973713  476942 ovs.go:35] 100.64.0.118 network not ready after 198 ping to gateway 100.64.0.1                                                                                                                                                         │
│ cni-server W0610 19:39:36.991756  476942 ovs.go:45] 100.64.0.118 network not ready after 200 ping, gw 100.64.0.1                                                                                                                                                                │
│ cni-server E0610 19:39:36.991800  476942 ovs_linux.go:488] failed to init ovn0 check: no packets received from gateway 100.64.0.1                                                                                                                                               │
│ cni-server E0610 19:39:36.991841  476942 klog.go:10] "failed to initialize node gateway" err="no packets received from gateway 100.64.0.1"
```

In this case, IP `100.64.0.1` belonged to our _Kube-OVN_ join network.

The fix described below likely serves as a general method of resolving pod
communication problems with the _join_ network without rebooting the affected
node.

# Repairing pod communication problems with the _join_ network

## The join network

The _Kube-OVN_ documentation says of the _join_ network [here](https://kubeovn.github.io/docs/v1.12.x/en/guide/subnet/#join-subnet):

> In the Kubernetes network specification, it is required that Nodes can
> communicate directly with all Pods. To achieve this in Overlay network mode,
> Kube-OVN creates a join Subnet and creates a virtual NIC ovn0 at each node
> that connect to the join subnet, through which the nodes and Pods can
> communicate with each other.

## `kubectl ko` plugin for OVS commands

The example below uses the `ko` plugin for `kubectl` which allows for issuing
OVS commands without explicitly using a `kubectl exec` to run commands in the
`ovs-ovn` pod, amongst other things. The _Genestack_ documentation shows
installation of this tool in
[docs/k8s-tools.md under heading "Install the ko plugin"](https://github.com/rackerlabs/genestack/blob/main/docs/k8s-tools.md#install-the-ko-plugin)

As an alternative to using `kubectl ko`, you can do something like
`kubectl -n kube-system exec -it <ovs-ovn pod for the node>`
and run, for example `ovs-vsctl` instead of `kubectl ko vsctl <node>`, etc.

## Resolving the issue

We rebooted one node to resolve this issue, but had the problem with 5 or 6
nodes, and continued trying to resolve the issue without rebooting. We managed
to do so as follows.

As an overview, this simply involved deleting the `ovn0` port for the node and
restarting the `kube-ovn-cni` pod for the node, but a more detailed example
follows.

1. Get the node name for the pod with the problem

    This command will clearly show the node name:

    ```
    kubectl -n kube-system get pod -o wide | grep kube-ovn-cni
    ```

    Example output:

    ```
    root@hyperconverged-0:~# kubectl -n kube-system get pod -o wide | grep kube-ovn-cni
    kube-ovn-cni-fhgh7                                       1/1     Running   0             11m   192.168.100.67    hyperconverged-0.cluster.local   <none>           <none>
    kube-ovn-cni-mj2hx                                       1/1     Running   0             11m   192.168.100.231   hyperconverged-1.cluster.local   <none>           <none>
    kube-ovn-cni-vjj7l                                       1/1     Running   0             11m   192.168.100.137   hyperconverged-2.cluster.local   <none>           <none>
    ```

    I will use a hyperconverged lab as created by _Genestack_'s
    `scripts/hyperconverged-lab.sh` for this example, and node
    `hyperconverged-1.cluster.local`, pod `kube-ovn-cni-mj2hx`

2. Check the logs

    ```
    kubectl -n kube-system logs kube-ovn-cni-mj2hx
    ```

    For a `kube-ovn-cni` pod with this problem, you see log messages like those
    shown above:

    ```
    W0610 21:36:34.676655  514300 ovs.go:35] 100.64.0.12 network not ready after 33 ping to gateway 100.64.0.1
    W0610 21:36:37.677050  514300 ovs.go:35] 100.64.0.12 network not ready after 36 ping to gateway 100.64.0.1
    W0610 21:36:40.676623  514300 ovs.go:35] 100.64.0.12 network not ready after 39 ping to gateway 100.64.0.1
    ```

3. Confirm the affected network as the _join_ network

    You can run `kubectl get subnet` and confirm the problem as with the `join`
    network or subnet, and see the gateway IP:

    ```
    root@hyperconverged-0:~# kubectl get subnet
    NAME          PROVIDER   VPC           PROTOCOL   CIDR            PRIVATE   NAT     DEFAULT   GATEWAYTYPE   V4USED   V4AVAILABLE   V6USED   V6AVAILABLE   EXCLUDEIPS       U2OINTERCONNECTIONIP
    join          ovn        ovn-cluster   IPv4       100.64.0.0/16   false     false   false     distributed   3        65530         0        0             ["100.64.0.1"]
    ovn-default   ovn        ovn-cluster   IPv4       10.236.0.0/14   false     true    true      distributed   112      262029        0        0             ["10.236.0.1"]
    ```

    The example output shows the join network with the gateway `100.64.0.1`

3. Try the ping

    Generally, the _Kube-OVN_ image has the ping command. This should also work
    on any pod with the `ping` command and serves as a way to confirm the
    problem. (However, some pods could probably restrict this with k8s security
    contexts and so forth, but most pods don't usually run in a way that
    prevents ping)

    ```
    kubectl -n kube-system exec kube-ovn-cni-mj2hx -c cni-server -- ping -c5 100.64.0.1
    ```

    Output:

    ```
    root@hyperconverged-0:~# kubectl -n kube-system exec kube-ovn-cni-mj2hx -c cni-server -- ping -c5 100.64.0.1
    PING 100.64.0.1 (100.64.0.1): 56 data bytes
    64 bytes from 100.64.0.1: icmp_seq=0 ttl=254 time=0.272 ms
    64 bytes from 100.64.0.1: icmp_seq=1 ttl=254 time=0.496 ms
    64 bytes from 100.64.0.1: icmp_seq=2 ttl=254 time=0.278 ms
    64 bytes from 100.64.0.1: icmp_seq=3 ttl=254 time=0.304 ms
    64 bytes from 100.64.0.1: icmp_seq=4 ttl=254 time=0.317 ms
    --- 100.64.0.1 ping statistics ---
    5 packets transmitted, 5 packets received, 0% packet loss
    round-trip min/avg/max/stddev = 0.272/0.333/0.496/0.083 ms
    root@hyperconverged-0:~#
    ```

    In this example, the ping works and gives output as shown, but for the
    example, I will go ahead with deleting the `ovn0` port in later steps as if
    it had not worked and I want to apply this fix.

4. Confirm the port exists

    ```
    root@hyperconverged-0:~# kubectl ko vsctl hyperconverged-1.cluster.local show | grep ovn0
            Port ovn0
                Interface ovn0
    ```

    Obviously, you could use many more commands to check this, such as
    `kubectl ko vsctl hyperconverged-1.cluster.local list-ports br-int | grep ovn0`,
    but that doesn't matter for this example.

5. Delete the port

    ```
    kubectl ko vsctl hyperconverged-1.cluster.local del-port br-int ovn0
    ```

    This does have a tendency to cause the `kube-ovn-cni` pod to restart by
    itself, and for the `ovn0` to get recreated automatically. If you delay
    confirming the port as deleted as in the previous step, you may find that
    the port already exists again and the `kube-ovn-cni` container already
    restarted.

6. Restart the `kube-ovn-cni` pod for the affected node

    ```
    root@hyperconverged-0:~# kubectl -n kube-system get pod -o wide | grep kube-ovn-cni | grep hyperconverged-1.cluster.local
    kube-ovn-cni-mj2hx                                       1/1     Running   4 (2m50s ago)   17h   192.168.100.231   hyperconverged-1.cluster.local   <none>           <none>
    root@hyperconverged-0:~#
    ```

    As mentioned above, you may find this pod automatically got restarted. If
    so, you can check the logs and possibly go ahead with the delete. If not,
    you can simply proceed to delete the pod:

    ```
    kubectl -n kube-system delete pod kube-ovn-cni-mj2hx
    ```

The above steps generally resolved this problem. You can follow through with checking by reading the logs of the `kube-ovn-cni` container and running the ping again to see if the pod could then actually ping the gateway for the _join_ network.

In a few cases, we had problems deleting the `kube-ovn-cni` pod, so a section on that follows.

### Problems deleting the `kube-ovn-cni` pod

As mentioned, in some cases, we had problems deleting the `kube-ovn-cni` pod.
Generally, when this happened, we could proceed to doing the `kubectl delete
pod` with the `--force` switch. Then, in some cases after using `--force`, this
would show errors in `kubectl logs` referring to the sandbox errors, and/or an
error binding to a port already in use when k8s tried to recreate the pod. When
that happened, we could generally resolve these by SSHing to the affected node
and killing the `kube-ovn-daemon` process as shown in the example below:

```
ubuntu@hyperconverged-1:~$ ps fauwxx | grep -i '[k]ube-ovn-daemon'
root      676519  0.5  0.4 1307276 79068 ?       Sl   16:35   0:43      \_ ./kube-ovn-daemon --ovs-socket=/run/openvswitch/db.sock --bind-socket=/run/openvswitch/kube-ovn-daemon.sock --enable-mirror=false --mirror-iface=mirror0 --node-switch=join --encap-checksum=true --service-cluster-ip-range=10.233.0.0/18 --iface=enp3s0 --dpdk-tunnel-iface=br-phy --network-type=geneve --default-interface-name=enp3s0 --cni-conf-dir=/etc/cni/net.d --cni-conf-file=/kube-ovn/01-kube-ovn.conflist --cni-conf-name=01-kube-ovn.conflist --logtostderr=false --alsologtostderr=true --log_file=/var/log/kube-ovn/kube-ovn-cni.log --log_file_max_size=200 --enable-metrics=true --kubelet-dir=/var/lib/kubelet --enable-tproxy=false --ovs-vsctl-concurrency=100 --secure-serving=false
ubuntu@hyperconverged-1:~$ sudo kill 676519
```

Then you can delete the `kube-ovn-cni` pod (if it doesn't simply restart by
itself from killing this process, which it tends to do, as also tends to happens
when deleting the `ovn0` port) and the pod should generally actually get a
normal `Running` state
