---
date: 2024-11-05
title: Running MetalLB on OpenStack Flex
authors:
  - cloudnull
description: >
  Running MetalLB on OpenStack Flex
categories:
  - Kubernetes
  - Authentication
---

# Running MetalLB on OpenStack Flex

![alt text](assets/images/2024-11-05/metallb-logo.png){ align=left : style="max-width:150px;background-color:rgb(28 144 243);" }

MetalLb is a load balancer for Kubernetes that provides a network load balancer implementation for Kubernetes clusters. MetalLB is a Kubernetes controller that watches for services of type `LoadBalancer` and provides a network load balancer implementation. The load balancer implementation is based on standard routing protocols. In this post we'll setup a set of allowed address pairs on the OpenStack Flex network to allow MetalLB to assign floating IPs to the load balancer service.

<!-- more -->

### Environment Setup

!!! note "This blog post was written with the following environment assumptions already existing"

    - Network: `tenant-net`

## Foundation

This guide assumes there is an operational Kubernetes cluster running on OpenStack Flex. To support this requirement, this guide will assume that the Kubernetes cluster is running following the Talos guide, which can be found [here](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex).

All operations will start from our Jump Host, which is a Debian instance running on OpenStack Flex adjacent to the Talos cluster. The Jump Host will be used to deploy Teleport to our Kubernetes cluster using Helm.

!!! note

    The jump host referenced within this guide will use the following variable, `${JUMP_PUBLIC_VIP}`, which is assumed to contain the public IP address of the node.

### Prerequisites

Before we begin, we need to ensure that we have the following prerequisites in place:

- An OpenStack Flex project with a Kubernetes cluster
- A working knowledge of Kubernetes

## Create new allowed address pairs

To allow MetalLB to assign floating IPs to the load balancer service, we need to create a set of allowed address pairs on the OpenStack Flex network. The allowed address pairs will allow the MetalLB pods to assign floating IPs to the load balancer service.

1. Create a new port.

``` shell
METAL_LB_IP=$(openstack --os-cloud default port create --network tenant-net metallb-vip-0 -f json | jq -r '.fixed_ips[0].ip_address')
```

!!! note "A word about port security"

    The port security group will be set to the default security group of the network. If security group changes are needed, set the `--security-group` flag when running to the `port create` command.

2. Associate the addressed assigned to the port as an allowed address pairs of our worker nodes.

``` shell
WORKER_0_PORT=$(openstack --os-cloud default port list --server talos-worker-0 -c ID -f value)
openstack --os-cloud default port set --allowed-address ip-address=${METAL_LB_IP} ${WORKER_0_PORT}

WORKER_1_PORT=$(openstack --os-cloud default port list --server talos-worker-1 -c ID -f value)
openstack --os-cloud default port set --allowed-address ip-address=${METAL_LB_IP} ${WORKER_1_PORT}

WORKER_2_PORT=$(openstack --os-cloud default port list --server talos-worker-2 -c ID -f value)
openstack --os-cloud default port set --allowed-address ip-address=${METAL_LB_IP} ${WORKER_2_PORT}
```

!!! note

    The IP address ${METAL_LB_IP} is an example. The port create process will assign a free IP address from the supplied network.
    Additionally it is possible to have multiple IP addresses in the allowed address pairs. Repeat the above steps for each IP address that will be used within MetalLB.

## Create The MetalLB Namespace

``` shell
kubectl create namespace metallb-system
```

Set the namespace security policy.

``` shell
kubectl label --overwrite namespace metallb-system \
        pod-security.kubernetes.io/enforce=privileged \
        pod-security.kubernetes.io/enforce-version=latest \
        pod-security.kubernetes.io/warn=privileged \
        pod-security.kubernetes.io/warn-version=latest \
        pod-security.kubernetes.io/audit=privileged \
        pod-security.kubernetes.io/audit-version=latest
```

### Add the Teleport Helm Repository

``` shell
helm repo add metallb https://metallb.github.io/metallb
```

Now update the repository:

``` shell
helm repo update
```

Run the following command to install MetalLB

``` shell
helm upgrade --install --namespace metallb-system metallb metallb/metallb
```

## Gather Node Information

``` shell
kubectl get nodes -o wide
```

!!! example "The output should look like this"

    ``` shell
    NAME                    STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION   CONTAINER-RUNTIME
    talos-control-plane-0   Ready    control-plane   19h   v1.31.2   10.0.0.208    <none>        Talos (v1.8.2)   6.6.58-talos     containerd://2.0.0-rc.6
    talos-control-plane-1   Ready    control-plane   19h   v1.31.2   10.0.0.60     <none>        Talos (v1.8.2)   6.6.58-talos     containerd://2.0.0-rc.6
    talos-control-plane-2   Ready    control-plane   19h   v1.31.2   10.0.0.152    <none>        Talos (v1.8.2)   6.6.58-talos     containerd://2.0.0-rc.6
    talos-worker-0          Ready    <none>          19h   v1.31.2   10.0.0.145    <none>        Talos (v1.8.2)   6.6.58-talos     containerd://2.0.0-rc.6
    talos-worker-1          Ready    <none>          19h   v1.31.2   10.0.0.235    <none>        Talos (v1.8.2)   6.6.58-talos     containerd://2.0.0-rc.6
    talos-worker-2          Ready    <none>          19h   v1.31.2   10.0.0.242    <none>        Talos (v1.8.2)   6.6.58-talos     containerd://2.0.0-rc.6
    ```

MetalLB will be installed on the "worker" nodes in the Kubernetes cluster. In this example, `talos-worker-0`, `talos-worker-1`, and `talos-worker-2`.

## Install MetalLB

To install MetalLB, we will use the following manifest

!!! example "talos-metallb.yaml"

    ``` yaml
    ---
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: openstack-external
      namespace: metallb-system
    spec:
      addresses:
        - ${METAL_LB_IP}/32  # The addresses listed here must match the same address in the allowed address pairs
      autoAssign: true  # Automatically assign an IP address from the pool to a service of type LoadBalancer
    ---
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: openstack-external-advertisement
      namespace: metallb-system
    spec:
      ipAddressPools:
        - openstack-external
      nodeSelectors:
        - matchLabels:
            kubernetes.io/hostname: talos-worker-0
        - matchLabels:
            kubernetes.io/hostname: talos-worker-1
        - matchLabels:
            kubernetes.io/hostname: talos-worker-2
    ```

!!! note "about autoAssign"

    Setting `autoAssign` to `true` allows MetalLB to automatically assign an IP address from the pool to a `LoadBalancer` service.

    Setting the `autoAssign` field to `false` will require operators to manually assign IP space to services. To assign the IP space manually, set an annotation in the service object, `metallb.universe.tf/address-pool`, with the value of the IP address or the name of the pool where the IP address will come from.

    !!! example "Example of manual assignment"

        ``` yaml
        annotations:
          metallb.universe.tf/address-pool: openstack-external
        ```

Deploy the MetalLB manifest

``` shell
kubectl apply -f talos-metallb.yaml
```

## Validate MetalLB

After deployment validate the MetalLB installation by running simple commands.

``` shell
kubectl --namespace metallb-system get ipaddresspools.metallb.io
```

!!! example "The output should look like this"

    ``` shell
    NAME                 AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
    openstack-external   true          false             ["${METAL_LB_IP}/32"]
    ```
