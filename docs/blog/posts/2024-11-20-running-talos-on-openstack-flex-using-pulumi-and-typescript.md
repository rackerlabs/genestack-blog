---
title: Deploy a Fully Automated Talos Cluster in Under 180 Seconds with Pulumi TypeScript
date: 2024-11-20
authors:
  - devx
description: >
  Deploy a Fully Automated Talos Cluster in Under 180 Seconds with Pulumi TypeScript
categories:
  - Kubernetes
  - DevOps
  - Pulumi
  - Automation
---

# Deploy a Fully Automated Talos Cluster in Under 180 Seconds with Pulumi TypeScript

![pulumi](assets/images/2024-11-15/pulumi.png){ align=left : style="max-width:125px" }
![talos-linux](assets/images/2024-11-04/talos-logo.png){ align=left }

Talos is a modern operating system designed for Kubernetes, providing a secure and minimal environment for running your clusters. Deploying Talos on OpenStack Flex can be streamlined using Pulumi, an infrastructure as code tool that allows you to define cloud resources using familiar programming languages like TypeScript.

In this guide, we'll walk through setting up the necessary network infrastructure on OpenStack Flex using Pulumi and TypeScript, preparing the groundwork for running Talos.

<!-- more -->

In earlier blog posts, Kevin demonstrated how to manually create a [Kubernetes cluster using Talos](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex/). Around the same time, I was exploring Talos and Pulumi using the Python language. However, I've been looking for an opportunity to dive into TypeScript, so I decided to give it a try. I have to say, I really enjoyed using TypeScript. Perhaps I'll share more of my thoughts on it in a future post, but for now, I'll mention that enjoyed learning about Typescript and probably continue to use it to develop.

Let's get started!

## Prerequisites
The following is what you will need to follow along.

  - An OpenStack account with appropriate permissions.
  - Pulumi (You can read my previous article on [getting started with pulumi](https://blog.rackspacecloud.com/blog/2024/11/08/getting_started_with_pulumi_and_openstack_flex/).)
  - Basic knowledge of programming (we'll use Typescript in this guide).
  - Node.js and pnpm: Ensure you have Node.js and npm installed on your machine.
  - Git installed (optional but recommended).
  - Talos Image in openstack (see the [Create the Image](https://blog.rackspacecloud.com/blog/2024/11/04/running_talos_on_openstack_flex/) in Kevin's post.)


## Setting Up the Pulumi Project
Now that we have the prerequisits out of the way. Let's set up a new Pulumi project for our new Talos Linux cluster.
```bash
# Create a new directory for the project
mkdir talos-cluster-ts
cd talos-cluster-ts

# Initialize a new Pulumi TypeScript project
pulumi new openstack-typescript -y

# Install the Talos pulumi Provider
pnpm i @pulumiverse/talos

```
!!! tip "This will create a stack called dev."


##  Update your Pulumi.yaml to utilize your OpenStack flex `clouds.yaml`
Edit your `Pulumi.yaml` file and add the following:
```yaml
  openstack:cloud:
    value: <replace with cloud>
```
Make sure you replace with your cloud value. In my config my cloud is called rxt.
The final file should look something like:
```yaml
name: talos-cluster-ts
description: A minimal OpenStack TypeScript Pulumi program
runtime:
  name: nodejs
  options:
    packagemanager: npm
config:
  pulumi:tags:
    value:
      pulumi:template: openstack-typescript
  openstack:cloud:
    value: rxt
```

## Update your index.ts
In your directory you should have a file called index.ts. Update the file the following:
```typescript
/*
 * Copyright (C) 2024 DevX <victor.palma@rackspace.com>
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, either version 3 of the License, or
 * (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <https://www.gnu.org/licenses/>.
 */

import * as pulumi from "@pulumi/pulumi";
import * as os from "@pulumi/openstack";
import * as talos from "@pulumiverse/talos";


const config = new pulumi.Config();

const clusterName = config.require("clusterName");
const tenantSubnetCIDR = config.require("tenantSubnetCIDR");
const serverFlavor = config.require("bastion_server_flavor");
const imageName = config.require("bastion_image_name");

// Get External Network
const externalNetworkName = config.require("externalNetworkName");
const extNet = pulumi.output(
  os.networking.getNetwork({
        name: externalNetworkName,
    })
);

// create the public keypair
const ssh_public_key = config.require("ssh_public_key");

// Create a key pair for SSH access
const keypair = new os.compute.Keypair(`${clusterName}-keypair`, {
    name: `${clusterName}-key`,
    publicKey: ssh_public_key,
});

// Create a Internal Tenant network
const tenantNetwork = new os.networking.Network(
    `${clusterName}-internal-tenant-network`,
    {
        name: `${clusterName}-internal-tenant-network`,
        adminStateUp: true,
        tags: [clusterName],
    }
);

// Create an internal tenant subnet within the newly created tenant network
const tenantSubnet = new os.networking.Subnet(
    `${clusterName}-tenant-subnet`,
    {
        name: `${clusterName}-tenant-subnet`,
        networkId: tenantNetwork.id,
        cidr: tenantSubnetCIDR,
        ipVersion: 4,
        dnsNameservers: ["8.8.8.8"],
        tags: [clusterName],
    }
);

// Create a router to connect the private network to the public network
const router = new os.networking.Router(`${clusterName}-router`, {
    adminStateUp: true,
    externalNetworkId: extNet.id,
    tags: [clusterName],
});

const routerInterface = new os.networking.RouterInterface(
    `${clusterName}-routerInterface`,
    {
        routerId: router.id,
        subnetId: tenantSubnet.id,
    }
);

//
// Security Group section and rules
//

// Create a security group
const secGroup = new os.networking.SecGroup(`${clusterName}-secGroup`, {
    name: `${clusterName}-secGroup`,
    description: `Security group for ${clusterName} control plane`,
    tags: [clusterName],
});

// Allow SSH Port 22
new os.networking.SecGroupRule(`${clusterName}-allow_22_port`, {
    direction: "ingress",
    ethertype: "IPv4",
    portRangeMax: 22,
    portRangeMin: 22,
    protocol: "tcp",
    remoteIpPrefix: "0.0.0.0/0",
    securityGroupId: secGroup.id,
});

new os.networking.SecGroupRule(`${clusterName}-allow_6443_port`, {
    direction: "ingress",
    ethertype: "IPv4",
    portRangeMax: 6443,
    portRangeMin: 6443,
    protocol: "tcp",
    remoteIpPrefix: "0.0.0.0/0",
    securityGroupId: secGroup.id,
});

new os.networking.SecGroupRule(`${clusterName}-allow_50000_port`, {
    direction: "ingress",
    ethertype: "IPv4",
    portRangeMax: 50000,
    portRangeMin: 50000,
    protocol: "tcp",
    remoteIpPrefix: "0.0.0.0/0",
    securityGroupId: secGroup.id,
});

new os.networking.SecGroupRule(`${clusterName}-allow_50001_port`, {
    direction: "ingress",
    ethertype: "IPv4",
    portRangeMax: 50001,
    portRangeMin: 50001,
    protocol: "tcp",
    remoteIpPrefix: "0.0.0.0/0",
    securityGroupId: secGroup.id,
});

const tcpIngressRule = new os.networking.SecGroupRule(`${clusterName}-tcpIngressRule`, {
    direction: "ingress",
    ethertype: "IPv4", // or "IPv6" if needed
    protocol: "tcp",
    securityGroupId: secGroup.id,
});

const udpIngressRule = new os.networking.SecGroupRule(`${clusterName}-udpIngressRule`, {
    direction: "ingress",
    ethertype: "IPv4", // or "IPv6" if needed
    protocol: "udp",
    securityGroupId: secGroup.id,
});


//
// Create a bastion server that will be used to interact with our Talos cluster
//

const bastionPort = new os.networking.Port(`${clusterName}-port`, {
    name: `${clusterName}-port`,
    networkId: tenantNetwork.id,
    fixedIps: [{ subnetId: tenantSubnet.id }],
    securityGroupIds: [secGroup.id],
    tags: [clusterName],
});

const bastionServer = new os.compute.Instance(`${clusterName}-bastion`, {
    name: `${clusterName}-bastion`,
    flavorName: serverFlavor,
    imageName: imageName,
    keyPair: keypair.name,
    availabilityZone: "nova",
    tags: [clusterName],
    networks: [{ port: bastionPort.id }],
    userData: `#!/bin/bash
apt-get update
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
mv kubectl /usr/local/bin
chmod +x /usr/local/bin/kubectl
curl -sL https://talos.dev/install | sh
`,
});

// Assign Floating IP to Load Balancer
const bastionFIP = new os.networking.FloatingIp(`${clusterName}-bastion_fip`,
  {
    description: clusterName,
    pool: externalNetworkName,
    portId: bastionPort.id,
    tags: [clusterName],
  }
);

export const bastion_ip = bastionFIP.address;
//
// Create Additional Network Infrastructure to deploy a talos linux cluster
//


// Create Load Balancer
const loadBalancer = new os.loadbalancer.LoadBalancer(`${clusterName}-lb`, {
  vipSubnetId: tenantSubnet.id,
  loadbalancerProvider: "ovn",
  tags: [clusterName],
});


// Create Listener on port 443
const talosControlPlaneListener = new os.loadbalancer.Listener(`${clusterName}-controlPlane-listener`, {
    name: `${clusterName}-controlPlane-listener`,
    loadbalancerId: loadBalancer.id,
    protocol: "TCP",
    protocolPort: 443,
    tags: [clusterName],
});

// Create Pool
const pool = new os.loadbalancer.Pool(`${clusterName}-controlPlane-pool`, {
    name: `${clusterName}-controlPlane-pool`,
    lbMethod: "SOURCE_IP_PORT",
    listenerId: talosControlPlaneListener.id,
    protocol: "TCP",
});

// Create Health Monitor
const healthMonitor = new os.loadbalancer.Monitor(`${clusterName}-controlPlane-health_monitor`, {
    poolId: pool.id,
    delay: 5,
    maxRetries: 4,
    timeout: 10,
    type: "TCP",
});

// Assign Floating IP to Load Balancer
const loadBalancerVIP = new os.networking.FloatingIp(`${clusterName}-loadBalancer-vip`,
  {
    description: clusterName,
    pool: externalNetworkName,
    portId: loadBalancer.vipPortId,
    tags: [clusterName],
  }
);

export const talosClusterIP: pulumi.Output<string> = pulumi.interpolate`https://${loadBalancerVIP.address}:443`;
//
// Create the talos cluster configuration
//

const talosSecrets = new talos.machine.Secrets("talos-secrets", {});


export const talosControlPlaneConfig = talos.machine.getConfigurationOutput({
    clusterName: clusterName,
    machineType: "controlplane",
    clusterEndpoint: talosClusterIP,
    machineSecrets: talosSecrets.machineSecrets,
    examples: true,
});


export const talosWorkerConfig = talos.machine.getConfigurationOutput({
    clusterName: clusterName,
    machineType: "worker",
    clusterEndpoint: talosClusterIP,
    machineSecrets: talosSecrets.machineSecrets,
});


//
// This section creates the Talos Linux Cluster
//


function createPort(
    serverName: string,
    tenantNetwork: pulumi.Input<string>,
    tenantSubnet: pulumi.Input<string>,
    secGroup: pulumi.Input<string>
): os.networking.Port {
    return new os.networking.Port(`${serverName}-port`, {
        name: `${serverName}-port`,
        networkId: tenantNetwork,
        fixedIps: [{ subnetId: tenantSubnet }],
        securityGroupIds: [secGroup],
    });
}

function createServer(
    serverName: string,
    port: os.networking.Port,
    keypair: pulumi.Input<string>,
    nodeType: string,
    serverFlavor: string,
    clusterName: pulumi.Input<string>,
): os.compute.Instance {
    // Read the user data from file
    let userDataFile = nodeType === "worker"
        ? talosWorkerConfig.machineConfiguration
        : talosControlPlaneConfig.machineConfiguration;
    const userData = userDataFile;

    return new os.compute.Instance(serverName, {
        imageName: "talos-1.8.2",
        name: serverName,
        flavorName: serverFlavor,
        keyPair: keypair,
        networks: [{ port: port.id }],
        userData: userData,
        tags: [clusterName],
    });
}

function updateLbMembers(
    server: os.compute.Instance,
    poolId: pulumi.Input<string>,
    serverName: string
): os.loadbalancer.Member {
    return new os.loadbalancer.Member(serverName, {
        poolId: poolId,
        address: server.accessIpV4,
        protocolPort: 6443,
    });
}

function createServers(
    tenantNetwork: pulumi.Input<string>,
    tenantSubnet: pulumi.Input<string>,
    extNet: pulumi.Input<string>,
    secGroup: pulumi.Input<string>,
    keypair: pulumi.Input<string>,
    nodeType: string,
    numNodes: number,
    serverFlavor: string,
): os.compute.Instance[] {
    const serverNames: string[] = [];
    for (let i = 1; i <= numNodes; i++) {
        serverNames.push(`${clusterName}_${nodeType}-${i}`);
    }

    const servers: os.compute.Instance[] = [];

    for (const serverName of serverNames) {
        const port = createPort(serverName, tenantNetwork, tenantSubnet, secGroup);
        const server = createServer(serverName, port, keypair, nodeType, serverFlavor, clusterName);
        servers.push(server);

        if (nodeType === "control_plane") {
            updateLbMembers(server, pool.id, serverName);
        }
    }

    return servers;
}

// Create Control Plane Servers
const controlPlaneServers = createServers(
    tenantNetwork.id,
    tenantSubnet.id,
    extNet.id,
    secGroup.id,
    keypair.id,
    "control_plane",
    3,
    "gp.0.4.8",
);

// Create Worker Servers
const workerServers = createServers(
    tenantNetwork.id,
    tenantSubnet.id,
    extNet.id,
    secGroup.id,
    keypair.id,
    "worker",
    3,
    "gp.0.4.8",
);

const talosContolPlaneNode = controlPlaneServers[0].accessIpV4;

const talosConfig = talos.client.getConfigurationOutput({
    clusterName: clusterName,
    clientConfiguration: talosSecrets.clientConfiguration,
    nodes: [ talosContolPlaneNode ],
    endpoints: [ talosContolPlaneNode ],
});

export const talosConfiguration = talosConfig.talosConfig

```
##  Add some variables our stack
The code above will create all the infrastructure needed. Don't worry if you forget to set the variables Pulumi will remind you and output something like:
```shell
Previewing update (dev):
     Type                 Name                  Plan       Info
 +   pulumi:pulumi:Stack  talos-cluster-ts-dev  create     1 error

Diagnostics:
  pulumi:pulumi:Stack (talos-cluster-ts-dev):
    error: Missing required configuration variable 'talos-cluster-ts:clusterName'
    	please set a value using the command `pulumi config set talos-cluster-ts:clusterName <value>`

```
This is simple to fix so just run the command it tells you with the corresponding value. for example:
```shell
pulumi config set talos-cluster-ts:clusterName talos-devx
```

Repeat this for the variables tenantSubnetCIDR, bastion_server_flavor, bastion_image_name, externalNetworkName, ssh_public_key. At the end of all those variables your file should look something like the following code block:
```yaml
encryptionsalt: v1:...............................
config:
  talos-cluster-ts:clusterName: talos-devx
  talos-cluster-ts:tenantSubnetCIDR: 192.168.100.0/24
  talos-cluster-ts:bastion_server_flavor: gp.0.1.2
  talos-cluster-ts:bastion_image_name: Debian-12
  talos-cluster-ts:externalNetworkName: PUBLICNET
  talos-cluster-ts:ssh_public_key: ssh-ed25519 .............................

```
Once that is done it's time to deploy.

## Deploying the Infrastructure
With all resources defined, you can now deploy the infrastructure:
```bash
pulumi up -y

```
Pulumi will create the network, subnet, router, security groups, key pair, and instances in your OpenStack Flex environment. This should take less than 2 minutes.

## Bootstraping the Talos cluster
Once it has finished running execute the follwing commands to copy the talosconfig file to the bastion server and start the bootstrap process.

```shell
pulumi stack output talosConfiguration --show-secrets > talosconfig

export BASTION=$(pulumi stack output bastion_ip)

# -o StricHostKeyChecking=no should not be done in prodution systems
scp -o StrictHostKeyChecking=no talosconfig debian@${BASTION}:~/

ssh debian@${BASTION} "talosctl --talosconfig talosconfig bootstrap"

# We sleep for about 50 seconds for the boostrap process to run
sleep 50
talosctl --talosconfig ./talosconfig get members

ssh debian@${BASTION} "talosctl --talosconfig talosconfig kubeconfig ~/.kube/config"
ssh debian@${BASTION} "kubectl get nodes"

```

All of this steps should take less than 3 minutes and the last command should show output something like the output below.

```
NAME                         STATUS     ROLES           AGE   VERSION
talos-devx-control-plane-1   NotReady   control-plane   25s   v1.31.1
talos-devx-control-plane-2   NotReady   <none>          19s   v1.31.1
talos-devx-control-plane-3   NotReady   control-plane   21s   v1.31.1
talos-devx-worker-1          NotReady   <none>          25s   v1.31.1
talos-devx-worker-2          NotReady   <none>          23s   v1.31.1
talos-devx-worker-3          NotReady   <none>          24s   v1.31.1
```

Click [here](https://github.com/devx/openstack-flex-examples/tree/main/pulumi/talos-cluster-ts) to got to my repo containing this example and other IaC examples.


## Conclusion
Using Pulumi and TypeScript to manage OpenStack resources offers a scalable and maintainable approach to infrastructure management. It enables version control, code reviews, and integration with CI/CD pipelines, enhancing collaboration and efficiency.

By following this guide, you've set up a foundational Talos kubernetes cluster on OpenStack Flex.

## Resources
  - [Pulumi OpenStack Provider Documentation](https://www.pulumi.com/registry/packages/openstack/)
  - [OpenStack Documentation](https://docs.openstack.org/)
  - [Pulumi Official Website](https://www.pulumi.com/)

