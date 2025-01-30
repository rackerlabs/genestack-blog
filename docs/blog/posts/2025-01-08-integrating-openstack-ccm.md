---
date: 2024-01-08
title: Create Octavia Loadbalancers dynamically with Kubernetes and Openstack Cloud Controller Manager
authors:
  - mikeruu
description: >
  Create Octavia Loadbalancers dynamically with Kubernetes and Openstack Cloud Controller Manager
categories:
  - Kubernetes
  - LoadBalancers
---

![octavia](assets/images/2025-01-08/octavia_logo.webp){ align=left : style="max-width:150px;background-color:rgb(28 144 243);" }

Load Balancers are essential in Kubernetes for exposing services to users in a cloud native way by distributing network traffic across multiple nodes, ensuring high availability, fault tolerance, and optimal performance for applications. 

By integrating with OpenStack’s Load Balancer as a Service (LBaaS) solutions like Octavia, Kubernetes can automate the creation and management of these critical resources with the use of the Openstack Cloud Controller Manager. The controller will identify services of type `LoadBalancer` and will automagically create cloud Loadbalancers on Openstack Flex with the Kubernetes nodes as members.

<!-- more -->

## Foundation

This guide assumes there is an operational Kubernetes cluster running on OpenStack Flex. To support this requirement, this guide will assume that the Kubernetes cluster is running following the Talos guide, which can be found [here](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex).

We have written blog posts about an alternative Loadbalancer solution for Kubernetes using [MetalLB on Openstack Flex](https://blog.rackspacecloud.com/blog/2024/11/05/running_metallb_on_openstack_flex). The main difference is that when using MetalLB you have to pre-provision a network port for the Floating IP, compared to [Openstack CCM](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md) where the Loadbalancer and Floating IP will be dyanically allocated with the option of creating an internal only Loadbalancer and foregoing the use of Floating IPs.

### Prerequisites

Before we begin, we need to ensure that we have the following prerequisites in place:

- An OpenStack Flex project with a Kubernetes cluster. If you dont already have one, we have a great post about creating a Kubernetes cluster with Talos [here](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex).
- A working knowledge of Kubernetes and Helm
- The Openstack CLI installed and configured to query Openstack Flex. [Here](https://blog.rackspacecloud.com/blog/2024/06/18/getting_started_with_rackspace_openstack_flex/) you can find a post that will walk you through the process.

## Gather the requirements

We will be using Helm to install the controller, so lets get the chart ready. This is the link to the chart [openstack-cloud-controller-manager](https://github.com/kubernetes/cloud-provider-openstack/tree/master/charts/openstack-cloud-controller-manager).

Add the Helm repository

``` shell
helm repo add cpo https://kubernetes.github.io/cloud-provider-openstack
helm repo update
```

#### Fill in the values file
This is an excerpt of the Helm values file that we will need to fill before we install the chart.

``` shell
cloudConfig:
  global:
    auth-url: "https://keystone.api.sjc3.rackspacecloud.com/v3/"
    username: "<USERNAME>"
    password: "<API-KEY>"
    domain-name: "rackspace_cloud_domain"
    tenant-name: "<TENANT>"
  networking:
  loadBalancer:
    floating-network-id: "<REPLACE>"
    floating-subnet-id: "<REPLACE>"
    network-id: "<REPLACE>"
    subnet-id: "<REPLACE>"
    create-monitor: true
    provider-requires-serial-api-calls: true
cluster:
  name: poc
```

We can gather the `global` settings from our clouds file used to authenticate the Openstack CLI against Openstack Flex used in the prerequisites.

``` shell
# cat ~/.config/openstack/clouds.yaml

clouds:
  flex-sjc3:
    auth:
      username: <REDACTED>
      project_name:  <REDACTED>
      password:  <REDACTED>
      auth_url: https://keystone.api.sjc3.rackspacecloud.com/v3/
      user_domain_name: rackspace_cloud_domain
      project_domain_name: rackspace_cloud_domain
    region: SJC3
    interface: public
    identity_api_version: 3

```

Next we can gather the `loadbalancer` details section by querying Openstack uinsg the CLI client:

Get the floating-network-id and floating-subnet-id from the shared public networks available in Openstack Flex.

``` shell
# openstack --os-cloud flex-sjc3 network list | grep PUBLICNET
+--------------------------------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| ID                                   | Name                                   | Subnets                                                                                                          |
+--------------------------------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| 723f8fa2-dbf7-4cec-8d5f-017e62c12f79 | PUBLICNET                              | 0e24addd-4b46-4dd8-9be3-08a351d724ad, 31bf7e05-be6e-4c5b-908d-abe47c80ba41, c093e1d4-51a1-48a4-ad30-920bd38dc1ed |

```

Next we can gather the IDs of the network and subnet used by the Kubernetes Cluster.

``` shell
# openstack --os-cloud flex-sjc3 network list | grep poc
+--------------------------------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| ID                                   | Name                                   | Subnets                                                                                                          |
+--------------------------------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| 07d6efa5-c4e4-420a-a897-daaa64d2ad96 | poc-k8s                            | e5eb87ec-4c20-4a6a-9082-db4b5d5caaab                                                                             |
```

### Additional fields

There are some additional fields on our values file that are required to make this work as expected, lets review them each:

``` shell
cloudConfig:
  global:
  networking:
  loadBalancer:
    create-monitor: true
    provider-requires-serial-api-calls: true
cluster:
  name: poc
```
#### Loadbalancer
- **create-monitor:** Indicates whether or not to create a health monitor for the service load balancer. A health monitor required for services that declare `externalTrafficPolicy: Local`. Default: false

- **provider-requires-serial-api-calls:** Some Octavia providers do not support creating fully-populated loadbalancers using a single [API Call](https://docs.openstack.org/api-ref/load-balancer/v2/?expanded=create-a-load-balancer-detail#creating-a-fully-populated-load-balancer). Setting this option to true will create loadbalancers using serial API calls which first create an unpopulated loadbalancer, then populate its listeners, pools and members. This is a compatibility option at the expense of increased load on the OpenStack API. Default: false

#### Cluster
- **name:** Resources created by the controller will have this added to their name for easy identification.

## Deploy the Helm chart

Now that we have gathered all the required information we are able to deploy the Helm chart.

``` shell
helm -n openstack-ccm upgrade --install openstack-ccm cpo/openstack-cloud-controller-manager --create-namespace --values openstack-ccm.yaml
```

#### Validate the chart installation

``` shell
kubectl -n openstack-ccm get pods
NAME                                       READY   STATUS    RESTARTS   AGE
openstack-cloud-controller-manager-j4fck   1/1     Running   0          11s
openstack-cloud-controller-manager-q7hrn   1/1     Running   0          11s
openstack-cloud-controller-manager-sf58c   1/1     Running   0          11s
```

## Deploy a Loadbalancer service

We will create a simple service of type `Loadbalancer` in order to have the controller create a cloud Loadbalancer in Openstack Flex.

!!! note
    We are also creating an nginx pod to have it listening on the port and pass the Loadbalancer health checks. Otherwise the LB will be in `operating_status = ERROR` in Openstack

``` shell
# cat nginx.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80       # Service port
    targetPort: 80 # Container port
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```
Create the service

``` shell
kubectl create -f nginx.yaml
```

After a few minutes we can verify that the service has been assiged an IP address `65.17.193.110` from the Floating IP network.

``` shell
 % kubectl get service
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
kubernetes      ClusterIP      10.53.0.1      <none>          443/TCP        70d
nginx-service   LoadBalancer   10.53.89.103   65.17.193.14    80:32179/TCP   103s
```

Review Octavia Loadbalancer
``` shell
 openstack --os-cloud flex-sjc3 loadbalancer list | grep poc
+--------------------------------------+--------------------------------------------------------------+----------------------------------+-----------------+---------------------+------------------+----------+
| id                                   | name                                                         | project_id                       | vip_address     | provisioning_status | operating_status | provider |
+--------------------------------------+--------------------------------------------------------------+----------------------------------+-----------------+---------------------+------------------+----------+
| 0c18797b-4c09-492f-80d6-28dddef6d9a9 | kube_service_poc_default_nginx-service                   | 372cea24da584d2787387263e25d1d97 | 172.20.20.208   | ACTIVE              | ONLINE           | amphora  |
+--------------------------------------+--------------------------------------------------------------+----------------------------------+-----------------+---------------------+------------------+----------+
```

Verify we can reach the service externally

``` shell
 curl -v 65.17.193.14
*   Trying 65.17.193.110:80...
* Connected to 65.17.193.110 (65.17.193.110) port 80 (#0)
> GET / HTTP/1.1
> Host: 65.17.193.110
> User-Agent: curl/8.1.1
> Accept: */*
```