---
date: 2024-11-04
title: Running CockroachDB on OpenStack Flex
authors:
  - cloudnull
description: >
  Running CockroachDB on OpenStack Flex
categories:
  - Kubernetes
  - Database
---

# Running CockroachDB on OpenStack Flex

![CockroachDB](assets/images/2024-11-04/cockroachlabs-logo.png){ align=left }
CockroachDB is a distributed SQL database that provides consistency, fault-tolerance, and scalability that has been purpose built for the cloud. In this guide, we will walk through deploying CockroachDB on an OpenStack Flex instance. As operators, we will need to create a new instance, install the CockroachDB software, and configure the service to run on the instance. The intent of this guide is to provide a simple functional example of how to deploy CockroachDB on an OpenStack Flex on Kubernetes.

<!-- more -->

## Foundation

This guide assumes there is an operational Kubernetes cluster running on OpenStack Flex. To support this requirement, this guide will assume that the Kubernetes cluster is running following the Talos guide, which can be found [here](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex).

An assumption of this guide is that the Kubernetes cluster has a working storage provider which can be used to create `PersistentVolumeClaims`. If the environment does not have a working storage provider, one will need to be deploy one before proceeding with this guide. In this guide, we will use Longhorn as our storage provider, which was deployed as part of the Talos on OpenStack Flex setup. Read more about Longhorn setup being used for this post [here](https://blog.rackspacecloud.com/blog/2024/11/04/running_longhorn_on_openstack_flex).

All operations will start from our Jump Host, which is a Debian instance running on OpenStack Flex adjacent to the Talos cluster. The Jump Host will be used to deploy Longhorn to our Kubernetes cluster using Helm.

!!! note

    The jump host referenced within this guide will use the following variable, `${JUMP_PUBLIC_VIP}`, which is assumed to contain the public IP address of the node.

### Prerequisites

Before we begin, we need to ensure that we have the following prerequisites in place:

- An OpenStack Flex project with a Kubernetes cluster
- A working knowledge of Kubernetes
- A working knowledge of Helm
- A working knowledge of OpenStack Flex
  - At least 180GiB of storage available to `PersistentVolumeClaims` (Longhorn)

!!! note

    This guide is using CockroachDB **1.7.2**, and the instructions may vary for other versions. Check the [CockroachDB documentation](https://www.cockroachlabs.com/whatsnew/) for the most up-to-date information on current releases.

Create a new namespace.

``` shell
kubectl create namespace cockroach-operator-system
```

Set the namespace security policy.

``` shell
kubectl label --overwrite namespace cockroach-operator-system \
        pod-security.kubernetes.io/enforce=privileged \
        pod-security.kubernetes.io/enforce-version=latest \
        pod-security.kubernetes.io/warn=privileged \
        pod-security.kubernetes.io/warn-version=latest \
        pod-security.kubernetes.io/audit=privileged \
        pod-security.kubernetes.io/audit-version=latest
```

## Install the CockroachDB Operator

Deploying the CockroachDB operator involves installing the CRDs and the operator itself.

### Deploy the CockroachDB CRDs

``` shell
kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/v2.15.1/install/crds.yaml
```

### Deploy the CockroachDB Operator

``` shell
kubectl --namespace cockroach-operator-system apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/v2.15.1/install/operator.yaml
```

``` shell
kubectl --namespace cockroach-operator-system get pods
```

!!! example "The output should look similar to the following"

    ``` shell
    NAME                                         READY   STATUS    RESTARTS   AGE
    cockroach-operator-manager-c8f97d954-5fwh4   1/1     Running   0          38s
    ```

### Deploy the CockroachDB Cluster

``` shell
kubectl --namespace cockroach-operator-system apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/v2.15.1/examples/example.yaml
```

!!! note "About the example cluster"

    This is a quick and easy cluster environment which is suitable for a wide range of purposes. However, for production use, administrators should consider a more robust configuration by reviewing this file and [CockroachDB documentation](https://www.cockroachlabs.com/docs/stable/).

#### Deploy the CockroachDB Client

Deploying the CockroachDB client is simple. It requires the installation of the client pod and the client secret.

``` shell
kubectl --namespace cockroach-operator-system create -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/v2.15.1/examples/client-secure-operator.yaml
```

``` shell
kubectl --namespace cockroach-operator-system exec -it cockroachdb-client-secure \
        -- ./cockroach sql \
        --certs-dir=/cockroach/cockroach-certs \
        --host=cockroachdb-public
```

!!! example "The above command will dropped into the SQL shell"

    ``` shell
    # Welcome to the CockroachDB SQL shell.
    # All statements must be terminated by a semicolon.
    # To exit, type: \q.
    #
    # Server version: CockroachDB CCL v24.2.3 (x86_64-pc-linux-gnu, built 2024/09/23 22:30:53, go1.22.5 X:nocoverageredesign) (same version as client)
    # Cluster ID: 162f3cf8-2699-4c59-b58d-a43afb34497c
    #
    # Enter \? for a brief introduction.
    #
    root@cockroachdb-public:26257/defaultdb>
    ```

    Running a simple `show databases;` command should return the following output.

    ``` shell
      database_name | owner | primary_region | secondary_region | regions | survival_goal
    ----------------+-------+----------------+------------------+---------+----------------
      defaultdb     | root  | NULL           | NULL             | {}      | NULL
      postgres      | root  | NULL           | NULL             | {}      | NULL
      system        | node  | NULL           | NULL             | {}      | NULL
    (3 rows)

    Time: 6ms total (execution 5ms / network 0ms)
    ```

## Conclusion

In this guide, we have walked through deploying CockroachDB on an OpenStack Flex instance on a Kubernetes cluster running Talos. We have also deployed the CockroachDB client and connected to the CockroachDB cluster to verify the deployment. This guide is intended to provide a simple example of how to deploy CockroachDB on an OpenStack Flex instance. For more information on CockroachDB, please refer to the [CockroachDB documentation](https://www.cockroachlabs.com/docs).
