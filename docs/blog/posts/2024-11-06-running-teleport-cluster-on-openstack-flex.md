---
date: 2024-11-06
title: Running Teleport Cluster on OpenStack Flex
authors:
  - cloudnull
description: >
  Running Teleport Cluster on OpenStack Flex
categories:
  - Kubernetes
  - Authentication
---

# Running Teleport Cluster on OpenStack Flex

![alt text](assets/images/2024-11-06/teleport-logo.png){ align=left }

Teleport is a modern security gateway for remotely accessing clusters of Linux servers via SSH or Kubernetes. In this guide, we will walk through deploying Teleport on an OpenStack Flex instance. As operators, we will need to create a new instance, install the Teleport software, and configure the service to run on the instance. This setup will allow us to access the Teleport web interface and create new users and roles, and manage access to the instance. The intent of this guide is to provide a simple example of how to deploy Teleport on an OpenStack Flex instance.

<!-- more -->

## Foundation

This guide assumes there is an operational Kubernetes cluster running on OpenStack Flex. To support this requirement, this guide will assume that the Kubernetes cluster is running following the Talos guide, which can be found [here](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex).

This guide also assumes that have metallb deployed on the Kubernetes cluster. If do not have metallb deployed, please refer to the [Running MetalLB on OpenStack Flex](https://blog.rackspacecloud.com/blog/2024/11/05/running-metallb-on-openstack-flex) guide. This guide will use the metallb service to route traffic to the Teleport service.

All operations will start from our Jump Host, which is a Debian instance running on OpenStack Flex adjacent to the Talos cluster. The Jump Host will be used to deploy Teleport to our Kubernetes cluster using Helm.

!!! note

    The jump host referenced within this guide will use the following variable, `${JUMP_PUBLIC_VIP}`, which is assumed to contain the public IP address of the node.

### Prerequisites

Before we begin, we need to ensure that we have the following prerequisites in place:

- An OpenStack Flex project with a Kubernetes cluster
- A working knowledge of Kubernetes
- A working knowledge of Helm
- A working knowledge of OpenStack Flex
- A working knowledge of PostgreSQL
- A working knowledge of Teleport

!!! note

    This guide is using Teleport **16.4.6**, and the instructions may vary for other versions. Check the [Teleport documentation](https://goteleport.com/docs/upcoming-releases/) for the most up-to-date information on current releases.

## Generate EC2 credentials

The following credentials will be used to authenticate the Teleport service to the S3 API provided by OpenStack Flex Object Storage.

``` shell
openstack --os-cloud default ec2 credentials create
```

!!! example "The output should look similar to the following"

    ``` shell
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | access     | ACCESS                           |
    | links      | {}                               |
    | project_id | PROJECT_ID                       |
    | secret     | SECRET                           |
    | trust_id   | None                             |
    | user_id    | USER_ID                          |
    +------------+----------------------------------+
    ```

Create an aws-config file.

``` shell
cat > ~/aws-config <<EOF
[plugins]
endpoint = awscli_plugin_endpoint

[profile default]
region = sjc3
s3 =
  endpoint_url = https://swift.api.sjc3.rackspacecloud.com
  signature_version = s3v4
s3api =
  endpoint_url = https://swift.api.sjc3.rackspacecloud.com
EOF
```

Create an aws-credentials file. Be sure to replace `ACCESS` and `SECRET` with the values from the previous command.

``` shell
cat > ~/aws-credentials <<EOF
[default]
aws_access_key_id = ACCESS
aws_secret_access_key = SECRET
EOF
```

## Create the Teleport Object Storage Bucket

This command, while using the `openstack` client, will create a new container in the Object Storage service. This container will be used to store the Teleport audit logs which will be used with the S3 compatible API.

``` shell
openstack --os-cloud default container create teleport-audit
```

## Security Group Setup

Ensure the project has a Security Group in place to allow traffic to our Teleport service. To create a Security Group, we will use the OpenStack CLI. Teleport will be using TLS Routing and will require the following ports to be open

Create a Talos security group, this `teleport-secgroup` will be used to permit Talos control plane and kubernetes traffic within the cluster.

``` shell
openstack --os-cloud default security group create teleport-secgroup
```

The security group will be used to permit traffic to the control plane nodes. We will open the following ports:

| Port | Protocol | Description |
|------|----------|-------------|
| 443  | TCP      | TLS Port    |
| 3080 | TCP      | APLN Port   |

The security group will be used to permit traffic to the control plane nodes. We will open the following ports:

``` shell
openstack --os-cloud default security group rule create --ingress --protocol tcp --dst-port 443 teleport-secgroup
openstack --os-cloud default security group rule create --ingress --protocol tcp --dst-port 3080 teleport-secgroup
openstack --os-cloud default security group rule create --ingress --protocol tcp teleport-secgroup
openstack --os-cloud default security group rule create --ingress --protocol udp teleport-secgroup
```

Additional rules can be added as needed for the environment. Refer to the Teleport Network Connectivity [documentation](https://goteleport.com/docs/reference/networking) for more information on additional ports and protocols that may be needed for the environment.

### Add worker nodes to the security group

The security group will need to be added to the worker nodes to allow traffic to the Teleport service.

``` shell
WORKER_0_PORT_ID=$(openstack --os-cloud default port list --server  talos-worker-0 -f value -c id)
openstack --os-cloud default port set --security-group teleport-secgroup ${WORKER_0_PORT_ID}

WORKER_1_PORT_ID=$(openstack --os-cloud default port list --server  talos-worker-1 -f value -c id)
openstack --os-cloud default port set --security-group teleport-secgroup ${WORKER_1_PORT_ID}

WORKER_2_PORT_ID=$(openstack --os-cloud default port list --server talos-worker-2 -f value -c id)
openstack --os-cloud default port set --security-group teleport-secgroup ${WORKER_2_PORT_ID}
```

Show the server details to verify the security group has been added.

``` shell
openstack --os-cloud default server show talos-worker-0
```

!!! example "Server output showing the security groups"

    ``` shell
    ...
    | security_groups                     | name='talos-secgroup'      |
    |                                     | name='teleport-secgroup'   |
    ...
    ```

## Deploying Teleport

To deploy Teleport, we will use Helm. Helm is a package manager for Kubernetes that allows us to define, install, and manage applications on our Kubernetes cluster. To install Helm, follow the instructions [here](https://helm.sh/docs/intro/install/).

### Installing Helm

``` shell
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Upon completion, Helm will be installed on the Jump Host.

### Add the Teleport Helm Repository

``` shell
helm repo add teleport https://charts.releases.teleport.dev
```

Now update the repository:

``` shell
helm repo update
```

### Create a new namespace for the Teleport cluster

``` shell
kubectl create namespace teleport-cluster
```

Set the namespace security policy.

``` shell
kubectl label --overwrite namespace teleport-cluster \
        pod-security.kubernetes.io/enforce=privileged \
        pod-security.kubernetes.io/enforce-version=latest \
        pod-security.kubernetes.io/warn=privileged \
        pod-security.kubernetes.io/warn-version=latest \
        pod-security.kubernetes.io/audit=privileged \
        pod-security.kubernetes.io/audit-version=latest
```

### Prepare the Database

Teleport requires a PostgreSQL database to store its configuration and audit logs. In this guide, we will use the Crunchy Postgres operator to deploy a Postgres cluster to our Kubernetes cluster. This guide will leverage the [Crunchty Postgres database deployment](https://blog.rackspacecloud.com/blog/2024/11/05/running_postgres_operator_from_crunchy_data_on_openstack_flex) post to provide for the database needs.

This section will assume the Crunchy Postgres Operator is running. The following examples will cover how to deploy a new Postgres Cluster within the `teleport-cluster` namespace using the online operator.

Create a new postgres cluster valules file which will be used to execute the helm based deployment.

!!! example "teleport-db-values.yaml"

    ``` yaml
    instanceReplicas: 3
    name: teleport-db
    instanceSize: 2Gi
    users:
      - name: teleport
        databases:
          - teleport_backend
          - teleport_audit
        options: 'LOGIN REPLICATION CREATEDB'
    ```

Create a new secrete for the db user, **teleport**

!!! example "teleport-db-secret.yaml"

    ``` yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: teleport-db-pguser-teleport
      labels:
        postgres-operator.crunchydata.com/cluster: teleport-db
        postgres-operator.crunchydata.com/pguser: teleport
    stringData:
      password: secrete  # replace with a real password
    ```

``` shell
kubectl --namespace teleport-cluster apply -f ~/teleport-db-secret.yaml
```

Run the deployment

!!! note "the helm command expect to be run from the crunchy postgres examples directory"

``` shell
helm upgrade --install --namespace teleport-cluster teleport-db helm/postgres \
             -f ~/teleport-db-values.yaml
```

### Create the Teleport Cluster Secrets

Create a kubernetes secret for the AWS credentials.

``` shell
kubectl --namespace teleport-cluster create secret generic aws-credentials \
        --from-file=credentials=aws-credentials \
        --from-file=config=aws-config
```

### Create the Teleport Cluster configuration

The following is an example of a Teleport configuration file. This file will be used to deploy the Teleport cluster using Helm. The configuration file will be used to define the cluster name, the DB connection string, and the S3 object storage URI.

!!! example "Create `teleport-values.yaml` file"

    For this to work, set the `clusterName` and the `acmeEmail` to suit the environment. Review the default [helm values file](https://github.com/gravitational/teleport/blob/master/examples/chart/teleport-cluster/values.yaml) for more information.

    ``` yaml
    chartMode: standalone
    clusterName: teleport.example.com  # replace with the preferred domain name
    persistence:
      enabled: false
    acme: true
    acmeEmail: "user@teleport.example.com"  # acmeEmail is the email address to provide during certificate registration (this is a Let's Encrypt requirement)
    auth:
      teleportConfig:
        teleport:
          log:
            severity: "INFO"
          storage:
            type: postgresql
            conn_string: postgres://teleport:secrete@teleport-db-primary.teleport-cluster.svc.cluster.local:5432/teleport_backend?sslmode=verify-ca&pool_max_conns=20
            # Uses the S3 API provided by OpenStack Flex Object Storage and the bucket created earlier
            audit_sessions_uri: "s3://teleport-audit?disablesse=true"
            ttl_job_cron: '*/20 * * * *'
      extraVolumes:
        - name: db-certs
          secret:
            secretName: teleport-db-cluster-cert
        - name: aws-creds
          secret:
            secretName: aws-credentials
      extraVolumeMounts:
        - name: db-certs
          mountPath: /root/.postgresql
          readOnly: true
        - name: aws-creds
          mountPath: /root/.aws
          readOnly: true
      extraEnv:
        - name: PGSSLROOTCERT
          value: /root/.postgresql/ca.crt
    ```

    Be sure to replace the database connection string with the correct connection string for the environment, with the correct username and password defined in the DB setup.

### Deploy the Teleport Cluster

Run the helm install command to deploy the Teleport cluster.

``` shell
helm upgrade --install teleport-cluster teleport/teleport-cluster \
     --create-namespace \
     --namespace=teleport-cluster \
     --version 16.4.6 \
     -f ~/teleport-values.yaml
```

Validate the teleport cluster deployment.

``` shell
kubectl --namespace teleport-cluster get services/teleport-cluster
```

## Create a Teleport Member

!!! example "teleport-member.yaml"

    ``` yaml
    kind: role
    version: v7
    metadata:
      name: member
    spec:
      allow:
        kubernetes_groups: ["system:masters"]
        kubernetes_labels:
          '*': '*'
        kubernetes_resources:
          - kind: '*'
            namespace: '*'
            name: '*'
            verbs: ['*']
    ```

``` shell
kubectl --namespace teleport-cluster exec -i deployment/teleport-cluster-auth -- tctl create -f < teleport-member.yaml
```

## Create a Teleport User

!!! tip "Remember to replace `YourUser` with the desired username"

``` shell
kubectl exec -ti deployment/teleport-cluster-auth -- tctl users add YourUser --roles=member,access,editor
```

The teleport service will use the loadbalancer type and assign an EXTERNAL-IP via metallb.

``` shell
kubectl --namespace teleport-cluster get services/teleport-cluster
```

!!! example "Teleport service output"

    ``` shell
    NAME               TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                                                                     AGE
    teleport-cluster   LoadBalancer   10.108.48.237   10.0.0.221        443:32567/TCP,3023:30573/TCP,3026:31708/TCP,3024:31129/TCP,3036:30414/TCP   86m
    ```

!!! note

    If the value is `<pending>` additional address pairs may be needed for the worker nodes. Refer to the [Running MetalLB on OpenStack Flex](https://blog.rackspacecloud.com/blog/2024/11/05/running-metallb-on-openstack-flex) guide to add additional allowed address pairs.

## Allowed Address Pairs

If the [MetalLB](https://blog.rackspacecloud.com/blog/2024/11/05/running-metallb-on-openstack-flex) environment is already configured with an allowed address pair, the IP address or port may need to be adjusted to match the expected port security or security group settings used for Teleport.

Validate the IP address port settings, and make appropriate changes.

``` shell
# ip-address is using the value of EXTERNAL-IP
openstack --os-cloud default port list --fixed-ip ip-address=10.0.0.221
```

!!! example

    ``` shell
    +--------------------------------------+---------------+-------------------+---------------------------------------------------------------------------+--------+
    | ID                                   | Name          | MAC Address       | Fixed IP Addresses                                                        | Status |
    +--------------------------------------+---------------+-------------------+---------------------------------------------------------------------------+--------+
    | bb2b010f-e792-4def-9350-e7a3944daee3 | metallb-vip-0 | fa:16:3e:85:16:9e | ip_address='10.0.0.221', subnet_id='b4448aa6-bb7d-4e01-86c1-80e589d3fb92' | DOWN   |
    +--------------------------------------+---------------+-------------------+---------------------------------------------------------------------------+--------+
    ```

Run the show command to get the port details.

``` shell
openstack --os-cloud default port show bb2b010f-e792-4def-9350-e7a3944daee3
```

!!! tip "If port security is `true` and the port security groups are not set appropriately, the security group settings may need to be adjusted"

    ``` shell
    openstack --os-cloud default port set --security-group teleport-secgroup bb2b010f-e792-4def-9350-e7a3944daee3
    ```

### Associate a Floating IP

If the IP address is not assigned to the port, and the Teleport Cluster will be accessed over the internet, associate a floating IP to the port.

``` shell
openstack --os-cloud default floating ip create --port bb2b010f-e792-4def-9350-e7a3944daee3 PUBLICNET
```

Retrieve and validate the floating IP address is associated with the port.

``` shell
PUBLIC_VIP=(openstack --os-cloud rxt-sjc-mine-flex floating ip list --fixed-ip 10.0.0.221 -f value -c "Floating IP Address")
```

## DNS Setup

A domain which will serve as the Teleport endpoint is needed. This guide assumes that record is `teleport.example.com` however, this value should be replaced with an actual domain.

> At this time, DNS will need to be managed outside of OpenStack Flex.

!!! example "The DNS record should point to the public IP address"

    The `${PUBLIC_VIP}` is a placeholder for the public IP address defined in the previous step.

    ``` txt
    ;; A     Records
    teleport.example.com.   1 IN  A      ${PUBLIC_VIP}

    ;; CNAME Records
    *.teleport.example.com. 1 IN  CNAME  teleport.example.com.
    ```

## Access the Teleport Environment

``` shell
tsh login --proxy=teleport.example.com --user=YourUser
```

In a browser, navigate to `https://teleport.example.com` to access the Teleport web interface.

![teleport-web](assets/images/2024-11-06/teleport-web.png)

## Conclusion

In this guide, we have walked through deploying Teleport on an OpenStack Flex environment where Talos is running. We installed the Teleport software, and configured the service to run on the cluster. This setup allows us to access the Teleport web interface, create new users and roles, and manage access to the instance. The intent of this guide is to provide a simple example of how to deploy Teleport on an OpenStack Flex via Kubernetes and highlight the flexibility of the environment. For more information on Teleport and running Teleport Agents, please refer to the [Teleport documentation](https://goteleport.com/docs/enroll-resources/agents/join-services-to-your-cluster/join-services-to-your-cluster/).
