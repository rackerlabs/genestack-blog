---
date: 2024-11-04
title: Running Longhorn on OpenStack Flex
authors:
  - cloudnull
description: >
  Running Longhorn on OpenStack Flex
categories:
  - Kubernetes
  - Storage
---

# Running Longhorn on OpenStack Flex

![Longhorn logo](assets/images/2024-11-04/longhorn-logo.png){ align=left : style="max-width:300px" }

Longhorn is a distributed block storage system for Kubernetes that is designed to be easy to deploy and manage. In this guide, we will walk through deploying Longhorn on an OpenStack Flex instance. As operators, we will need to create a new instance, install the Longhorn software, and configure the service to run on the instance. This setup will allow us to access the Longhorn web interface and create new volumes, snapshots, and backups. The intent of this guide is to provide a simple example of how to deploy Longhorn on an OpenStack Flex instance.

<!-- more -->

## Foundation

This guide assumes there is an operational Kubernetes cluster running on OpenStack Flex. To support this requirement, this guide will assume that the Kubernetes cluster is running following the Talos guide, which can be found [here](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex).

All operations will start from our Jump Host, which is a Debian instance running on OpenStack Flex adjacent to the Talos cluster. The Jump Host will be used to deploy Longhorn to our Kubernetes cluster using Helm.

!!! note

    The jump host referenced within this guide will use the following variable, `${JUMP_PUBLIC_VIP}`, which is assumed to contain the public IP address of the node.

### Prerequisites

Before we begin, we need to ensure that we have the following prerequisites in place:

- An OpenStack Flex project with a Kubernetes cluster
- A working knowledge of Kubernetes
- A working knowledge of Helm
- A working knowledge of OpenStack Flex
- A working knowledge of Longhorn

!!! note

    This guide is using Longhorn **1.7.2**, and the instructions may vary for other versions. Check the [Longhorn documentation](https://longhorn.io/docs/) for the most up-to-date information on current releases.

## Deploying storage volumes

Longhorn works by creating volumes that are attached to the Talos workers. These volumes are then used to store data for the applications running on the cluster. The first step in deploying Longhorn is to create a new volume that will be used to store data for the applications running on the cluster.

### Creating new volumes

Using the OpenStack CLI we can create new volumes by running the following commands

``` shell
openstack --os-cloud default volume create --type Capacity --size 100 longhorn-0
openstack --os-cloud default volume create --type Capacity --size 100 longhorn-1
openstack --os-cloud default volume create --type Capacity --size 100 longhorn-2
```

### Attaching volumes to the Talos workers

Now that we have created a new volume, we can attach it to the Talos workers using the OpenStack CLI. This will allow the volume to be used by the applications running on the cluster.

``` shell
openstack --os-cloud default server add volume talos-worker-0 longhorn-0
openstack --os-cloud default server add volume talos-worker-1 longhorn-1
openstack --os-cloud default server add volume talos-worker-2 longhorn-2
```

## Prepare the Volumes for Longhorn

Before we can deploy Longhorn, we need to prepare the volumes that we created in the previous step. To do this, we will need to format the volumes and mount them to the Talos workers.
Run a quick scan to pick up all the members within the cluster.

``` shell
talosctl --talosconfig ./talosconfig get members
```

!!! example "The Output Will Look Like This"

    ``` shell
    NODE         NAMESPACE   TYPE     ID                      VERSION   HOSTNAME                          MACHINE TYPE   OS               ADDRESSES
    10.0.0.208   cluster     Member   talos-control-plane-0   11        talos-control-plane-0.novalocal   controlplane   Talos (v1.8.2)   ["10.0.0.208"]
    10.0.0.208   cluster     Member   talos-control-plane-1   4         talos-control-plane-1.novalocal   controlplane   Talos (v1.8.2)   ["10.0.0.60"]
    10.0.0.208   cluster     Member   talos-control-plane-2   6         talos-control-plane-2.novalocal   controlplane   Talos (v1.8.2)   ["10.0.0.152"]
    10.0.0.208   cluster     Member   talos-worker-0          9         talos-worker-0.novalocal          worker         Talos (v1.8.2)   ["10.0.0.16"]
    10.0.0.208   cluster     Member   talos-worker-1          12        talos-worker-1.novalocal          worker         Talos (v1.8.2)   ["10.0.0.249"]
    10.0.0.208   cluster     Member   talos-worker-2          12        talos-worker-2.novalocal          worker         Talos (v1.8.2)   ["10.0.0.110"]
    ```

Here we can see the nodes and their related addresses. We will use this information to connect to the nodes and prepare the volumes.

Run a quick volume discovery to verify that our expected volume is connected to the node.

``` shell
talosctl --talosconfig ./talosconfig get discoveredvolumes --nodes 10.0.0.16
```

!!! example "The Output Will Look Like This"

    ``` shell
    NODE        NAMESPACE   TYPE               ID      VERSION   TYPE        SIZE     DISCOVERED   LABEL       PARTITIONLABEL
    10.0.0.16   runtime     DiscoveredVolume   loop2   1         disk        684 kB   squashfs
    10.0.0.16   runtime     DiscoveredVolume   loop3   1         disk        2.6 MB   squashfs
    10.0.0.16   runtime     DiscoveredVolume   loop4   1         disk        75 MB    squashfs
    10.0.0.16   runtime     DiscoveredVolume   vda     2         disk        43 GB    gpt
    10.0.0.16   runtime     DiscoveredVolume   vda1    1         partition   105 MB   vfat                     EFI
    10.0.0.16   runtime     DiscoveredVolume   vda2    1         partition   1.0 MB                            BIOS
    10.0.0.16   runtime     DiscoveredVolume   vda3    1         partition   982 MB   xfs          BOOT        BOOT
    10.0.0.16   runtime     DiscoveredVolume   vda4    1         partition   1.0 MB                            META
    10.0.0.16   runtime     DiscoveredVolume   vda5    2         partition   92 MB    xfs          STATE       STATE
    10.0.0.16   runtime     DiscoveredVolume   vda6    2         partition   42 GB    xfs          EPHEMERAL   EPHEMERAL
    10.0.0.16   runtime     DiscoveredVolume   vdb     1         disk        1.1 GB   swap
    10.0.0.16   runtime     DiscoveredVolume   vdc     1         disk        11 GB
    ```

!!! note

    The output will show all the volumes attached to the node. The volume we're looking for is **vdc** which is reporting an 11 GB size.

Armed with the information, we can see that the volume we're looking for is attached to the node. We can now format and mount the volume.

Create a patch file which will modify the Talos worker to mount the volume.

!!! example "talos-longhorn-disk.yaml"

    ``` yaml
    machine:
      kubelet:
        extraMounts:
          - destination: /var/lib/longhorn
            type: bind
            source: /var/lib/longhorn
            options:
              - bind
              - rshared
              - rw
      disks:
          - device: /dev/vdc
            partitions:
              - mountpoint: /var/lib/longhorn
    ```

Now update the Talos configuration to format the volume and mount it.

``` shell
talosctl --talosconfig ./talosconfig patch mc --patch @talos-longhorn-disk.json --nodes 10.0.0.16
```

!!! tip

    If all nodes have the same disk layout and the same volume attached, use the `--nodes` flag with comma separated values to apply the patch to all nodes at once.

    ``` shell
    --nodes 10.0.0.110,10.0.0.249,10.0.0.16
    ```

Once the patch has been applied the node will reboot, and the volume will be formatted and mounted. Validate the volume is mounted and formatted by rerunning the `discoveredvolumes` command.

``` shell
talosctl --talosconfig ./talosconfig get discoveredvolumes --nodes 10.0.0.16
```

!!! example "the output will looks like this"

    ``` shell
    NODE        NAMESPACE   TYPE               ID      VERSION   TYPE        SIZE     DISCOVERED   LABEL       PARTITIONLABEL
    10.0.0.16   runtime     DiscoveredVolume   loop2   1         disk        684 kB   squashfs
    10.0.0.16   runtime     DiscoveredVolume   loop3   1         disk        2.6 MB   squashfs
    10.0.0.16   runtime     DiscoveredVolume   loop4   1         disk        75 MB    squashfs
    10.0.0.16   runtime     DiscoveredVolume   vda     1         disk        43 GB    gpt
    10.0.0.16   runtime     DiscoveredVolume   vda1    1         partition   105 MB   vfat                     EFI
    10.0.0.16   runtime     DiscoveredVolume   vda2    1         partition   1.0 MB                            BIOS
    10.0.0.16   runtime     DiscoveredVolume   vda3    1         partition   982 MB   xfs          BOOT        BOOT
    10.0.0.16   runtime     DiscoveredVolume   vda4    1         partition   1.0 MB                            META
    10.0.0.16   runtime     DiscoveredVolume   vda5    1         partition   92 MB    xfs          STATE       STATE
    10.0.0.16   runtime     DiscoveredVolume   vda6    1         partition   42 GB    xfs          EPHEMERAL   EPHEMERAL
    10.0.0.16   runtime     DiscoveredVolume   vdb     1         disk        1.1 GB   swap
    10.0.0.16   runtime     DiscoveredVolume   vdc     1         disk        11 GB    gpt
    10.0.0.16   runtime     DiscoveredVolume   vdc1    3         partition   11 GB    xfs
    ```

## Deploying Longhorn

With the workers situated, we can now deploy Longhorn to the Kubernetes cluster. To do this, we will use Helm to install the Longhorn chart.

Add Longhorn to the Helm repos.

``` shell
helm repo add longhorn https://charts.longhorn.io
```

Update the repos.

``` shell
helm repo update
```

Create a new namespace for Longhorn.

``` shell
kubectl create namespace longhorn-system
```

Set the longhorn-system namespace security policy.

``` shell
kubectl label --overwrite namespace longhorn-system \
        pod-security.kubernetes.io/enforce=privileged \
        pod-security.kubernetes.io/enforce-version=latest \
        pod-security.kubernetes.io/warn=privileged \
        pod-security.kubernetes.io/warn-version=latest \
        pod-security.kubernetes.io/audit=privileged \
        pod-security.kubernetes.io/audit-version=latest
```

Install Longhorn.

``` shell
helm upgrade --install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.7.2
```

!!! tip

    For more information on all of the options that longhorn has to offer when deploying via helm, please refer to the [Longhorn documentation](https://longhorn.io/docs/1.7.2/advanced-resources/deploy/customizing-default-settings/#using-the-longhorn-deployment-yaml-file).

The deployment will take a few minutes, watch the nodes to validate the deployment is ready.

``` shell
kubectl -n longhorn-system get nodes.longhorn.io
```

!!! example "Healthy output will look like this"

    ``` shell
    NAME             READY   ALLOWSCHEDULING   SCHEDULABLE   AGE
    talos-worker-0   True    true              True          9m20s
    talos-worker-1   True    true              True          9m20s
    talos-worker-2   True    true              True          9m19s
    ```

Validate functionality by creating the volume test deployment.

``` shell
kubectl create -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/examples/pod_with_pvc.yaml
```

Assuming everything is working, the pod will spawn and the volume attach. Validate the volume is attached by running the following command.

``` shell
kubectl -n longhorn-system get volumes.longhorn.io
```

!!! example "Healthy output will look like this"

    ``` shell
    NAME                                       DATA ENGINE   STATE      ROBUSTNESS   SCHEDULED   SIZE         NODE   AGE
    pvc-95225b70-90de-4b77-b55f-0d4f089e8a07   v1            detached   unknown                  2147483648          3s
    ```

## Conclusion

Longhorn provides a robust storage solution that is container native and open infrastructure ready. By following this guide, the latest release of Longhorn will have been successfully deployed a Talos cluster running within OpenStack Flex instances. The next steps would be to explore the Longhorn documentation and experiment with the various features that Longhorn has to offer.
