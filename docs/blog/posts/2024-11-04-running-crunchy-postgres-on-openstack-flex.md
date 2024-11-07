---
date: 2024-11-05
title: Running Postgres Operator from Crunchy Data on OpenStack Flex
authors:
  - cloudnull
description: >
  Running Postgres Operator from Crunchy Data on OpenStack Flex
categories:
  - Kubernetes
  - Database
---

# Running Crunchydata Postgres on OpenStack Flex

![Crunchdata](assets/images/2024-11-05/crunchydata-logo.png){ align=left : style="max-width:125px" }

Crunchydata provides a Postgres Operator that simplifies the deployment and management of PostgreSQL clusters on Kubernetes. In this guide, we will walk through deploying the Postgres Operator from Crunchy Data on an OpenStack Flex instance. As operators, we will need to create a new instance, install the Postgres Operator software, and configure the service to run on the instance. The intent of this guide is to provide a simple functional example of how to deploy the Postgres Operator from Crunchy Data on an OpenStack Flex on Kubernetes.

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
  - At least 1GiB of storage available to `PersistentVolumeClaims` (Longhorn)

!!! note

    This guide is using Crunchydata **5.7**, and the instructions may vary for other versions. Check the [Crunchydata documentation](https://access.crunchydata.com/documentation/postgres-operator/latest) for the most up-to-date information on current releases.

## Create a New Namespace

``` shell
kubectl create namespace crunchy-operator-system
```

Set the namespace security policy.

``` shell
kubectl label --overwrite namespace crunchy-operator-system \
        pod-security.kubernetes.io/enforce=privileged \
        pod-security.kubernetes.io/enforce-version=latest \
        pod-security.kubernetes.io/warn=privileged \
        pod-security.kubernetes.io/warn-version=latest \
        pod-security.kubernetes.io/audit=privileged \
        pod-security.kubernetes.io/audit-version=latest
```

## Install the Crunchdata Postgres Operator

Before getting started, set a few environment variables that will be used throughout the guide.

``` shell
export CRUNCHY_OPERATOR_NAMESPACE=crunchy-operator-system
export CRUNCHY_CLUSTER_NAMESPACE=crunchy-operator-system  # This can be a different namespace
export CRUNCHY_CLUSTER_NAME=hippo
export CRUNCHY_DB_REPLICAS=3
export CRUNCHY_DB_SIZE=1Gi
```

Retrieve the operator helm chart and change into the directory.

``` shell
git clone https://github.com/CrunchyData/postgres-operator-examples
cd postgres-operator-examples
```

Install the operator helm chart.

``` shell
helm upgrade --install --namespace ${CRUNCHY_OPERATOR_NAMESPACE} crunchy-operator helm/install
```

## Create a Crunchydata Postgres Cluster

Create a helm overrides file for the database deployment. The file should contain the following information. Replace the `${CRUNCHY_DB_REPLICAS}`, `${CRUNCHY_CLUSTER_NAME}`, and `${CRUNCHY_DB_SIZE}` with the desired values for the deployment.

!!! example "crunchy-db.yaml"

    ``` yaml
    instanceReplicas: ${CRUNCHY_DB_REPLICAS}
    name: ${CRUNCHY_CLUSTER_NAME}
    instanceSize: ${CRUNCHY_DB_SIZE}
    users:
      - name: rhino
        databases:
          - zoo
        options: 'NOSUPERUSER'
    ```

Create a new secret for the user **rhino**

!!! example "crunchy-rhino-secret.yaml"

    ``` yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: ${CRUNCHY_CLUSTER_NAME}-pguser-rhino
      labels:
        postgres-operator.crunchydata.com/cluster: ${CRUNCHY_CLUSTER_NAME}
        postgres-operator.crunchydata.com/pguser: rhino
    stringData:
      password: river
    ```

``` shell
kubectl --namespace ${CRUNCHY_CLUSTER_NAMESPACE} apply -f crunchy-rhino-secret.yaml
```

Run the Deployment

``` shell
helm upgrade --install --namespace ${CRUNCHY_CLUSTER_NAMESPACE} hippo helm/postgres \
             -f crunchy-db.yaml
```

!!! tip

    Track the state of the deployment with the following

    ``` shell
    kubectl -n ${CRUNCHY_CLUSTER_NAMESPACE} get pods --selector=postgres-operator.crunchydata.com/cluster=${CRUNCHY_CLUSTER_NAME},postgres-operator.crunchydata.com/instance
    ```

## Verify the Crunchydata Postgres Cluster

``` shell
kubectl --namespace ${CRUNCHY_CLUSTER_NAMESPACE} get svc --selector=postgres-operator.crunchydata.com/cluster=${CRUNCHY_CLUSTER_NAME}
```

## Conclusion

In this guide, we have deployed the Crunchydata Postgres Operator on an OpenStack Flex Kubernetes cluster. We have also created a new Postgres cluster using the operator. This guide is intended to provide a simple functional example of how to deploy the Crunchydata Postgres Operator on an OpenStack Flex Kubernetes cluster. For more information on the Crunchydata Postgres Operator, please refer to the [Crunchydata documentation](https://access.crunchydata.com/documentation/postgres-operator/latest).
