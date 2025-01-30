---
title: Getting Started with Pulumi and OpenStack Flex
date: 2024-11-08
authors:
  - devx
description: >
  Getting Started with Pulumi and OpenStack Flex
categories:
  - Kubernetes
  - DevOps
  - Pulumi
  - Automation
---

# Getting Started with Pulumi and OpenStack

![pulumi](assets/images/2024-11-15/pulumi.png){ align=left : style="max-width:125px" }

Pulumi is an open-source infrastructure-as-code (IaC) platform that enables you to define, deploy, and manage cloud infrastructure using familiar programming languages like Python, JavaScript, TypeScript, Go, and C#. By leveraging your existing coding skills and knowledge, Pulumi allows you to build, deploy, and manage infrastructure on any cloud provider, including AWS, Azure, Google Cloud, Kubernetes, and OpenStack. Unlike traditional tools that rely on YAML files or domain-specific languages, Pulumi offers a modern approach by utilizing general-purpose programming languages for greater flexibility and expressiveness. This means you can use standard programming constructs—such as loops, conditionals, and functions—to create complex infrastructure deployments efficiently.


<!-- more -->


Infrastructure as Code (IaC) has revolutionized the way we manage and deploy cloud resources. Pulumi, a modern IaC tool, allows you to define cloud infrastructure using familiar programming languages. When combined with OpenStack, an open-source cloud computing platform, you gain unparalleled flexibility and control over your cloud environments. In this guide, we'll walk you through getting started with Pulumi and OpenStack. We will do so by creating an secure entry point (bastion server) and exposing securely by only allowing access to the server via ssh. 

## Prerequisites
  - An OpenStack account with appropriate permissions.
  - Basic knowledge of programming (we'll use Python in this guide).
  - Python installed on your machine (version 3.6 or higher).
  - Git installed (optional but recommended).

## What is Pulumi?
Pulumi is an open-source IaC tool that enables you to define and manage cloud resources using general-purpose programming languages like Python, JavaScript, TypeScript, Go, and C#. Unlike traditional templating languages or domain-specific languages (DSLs), Pulumi leverages the full power of these languages, including loops, conditions, and functions.

## Setting Up Your Environment
Before we dive in, ensure your environment is ready:

  - OpenStack Credentials: Obtain your OpenStack authentication credentials (e.g., `auth_url`, `project_name`, `username`, `password`, `region_name`).
  - Install Python: If you haven't installed Python yet, download it from the [official website](https://www.python.org/) and follow the installation instructions.
  - Install Git: While optional, Git is useful for version control. Download it from the [official website](https://git-scm.com/).

## Installing Pulumi
Pulumi provides a straightforward installation process you can follow the instructions bellow or you can go to the [official website](https://www.pulumi.com/docs/iac/download-install/):

### MacOS
The easiest way is to install via brew. For more information visit the [official website]().
```
brew install pulumi/tap/pulumi
```

### For linux utilize the Installer Script
Run the following command in your terminal or command prompt:

```bash
curl -fsSL https://get.pulumi.com | sh
```
This script downloads and installs the Pulumi CLI to ~/.pulumi/bin, adding it to your PATH. Don't forget to add `~/.pulumi/bin` to your PATH.

### Verifying the Installation
Check that Pulumi is installed correctly:
```bash
pulumi version
```
You should see the Pulumi version number printed out.

## Configuring Pulumi for OpenStack
Pulumi interacts with OpenStack through environment variables or a configuration file. If you have a working `clouds.yaml` config file then you can add it to your project as shown in step 3 in the Deploying resources to OpenStack Flex section.


If you do not have a working clouds config file. We'll use environment variables for simplicity.

### Setting Environment Variables
Set the following environment variables with your OpenStack credentials:
```bash
export OS_AUTH_URL="https://your-openstack-auth-url"
export OS_PROJECT_NAME="your-project-name"
export OS_USERNAME="your-username"
export OS_PASSWORD="your-password"
export OS_REGION_NAME="your-region-name"
```
Replace the placeholders with your actual credentials.

#### Optional Variables
You might also need to set additional variables like OS_USER_DOMAIN_NAME or OS_PROJECT_DOMAIN_NAME depending on your OpenStack setup.

## Creating Your First Pulumi Project
Now, let's create a Pulumi project to deploy resources to OpenStack.

### Step A: Initialize Pulumi's self-managed backend
For this demo we will *not* be using pulumi cloud. Instead we will use a self-managed backend. To keep things simple we will use our local filesystem. For more information on [managing backends and state visit the official site](https://www.pulumi.com/docs/iac/concepts/state-and-backends/).
```bash
pulumi login --local
```
You will see Logged into `<my-machine>` as `<my-user>` (file://~). This will result in the storing of all stacks created in a JSON format in the directory `~/.pulumi`.

### Step B: Create a New Directory
Create and navigate to a new directory for your project:
```bash
mkdir pulumi-openstack-flex-demo
cd pulumi-openstack-flex-demo
```

### Step C: Initialize the Pulumi Project
Run the Pulumi new command to bootstrap a new project utilizing all the python defaults. It will also ask you setup a passphrase to protect `config/secrets`.
```bash
pulumi new python -y
```

This command does the following:
  - Creates a new Pulumi project using the OpenStack Python template.
  - Installs necessary dependencies.
  - Prompts you for a project name, description, and stack name.
** Depending on your system you might be asked to install the python venv module.

### Step D: Review the Generated Files
The template generates several files:

  - `Pulumi.yaml`: The project metadata.
  - `__main__.py`: The main program where you'll define your infrastructure.
  - `requirements.txt`: Python dependencies.
  - `Pulumi.dev.yaml`: The Projects stack that was automatically created for you.
  - `venv`: This directory has a virtual python environment to help faciliate development.


!!! note

    Pulumi stacks are isolated instances of your infrastructure configurations that enable you to manage multiple environments—such as development, staging, and production—within the same project. By having many stacks for the same project, you can deploy the same infrastructure code with different settings or parameters tailored to each environment, ensuring consistent and efficient infrastructure management across all stages.

## Deploying Resources to OpenStack Flex
Let's modify `__main__.py` to deploy a simple resource, such as an OpenStack instance.

#### Step 0: Active the venv that was created
Activate the virtual environment

Fish Shell:
```bash
source venv/bin/activate.fish

```
Bash, sh, zsh Shells:
```bash
source venv/bin/activate
```

Depending on your shell configuration it might show you that you are now utilizing a virtual environment. You can also verify by doing the following:
```bash
which python
```
This will return something like 
```bash
/home/debian/pulumi-openstack-flex-demo/venv/bin/python
```

#### Step 1: Install the OpenStack Provider
Ensure the Pulumi OpenStack provider is installed:
```bash

pip install pulumi-openstack
```
#### Step 2: Update requirements.txt
Add pulumi-openstack to your requirements.txt:
```bash
pulumi
pulumi-openstack
```
Run `pip install -r requirements.txt` to install dependencies.

#### Step 3: Update your Pulumi.yaml to utilize your OpenStack flex `clouds.yaml`
Edit your `Pulumi.yaml` file and add the following:
```yaml
  openstack:cloud:
    value: <replace with cloud> 
```
Make sure you replace with your cloud value. In my config my cloud is called rxt. 
The final file should look something like:
```
name: pulumi-openstack-flex-demo
description: A minimal Python Pulumi program
runtime:
  name: python
  options:
    toolchain: pip
    virtualenv: venv
config:
  pulumi:tags:
    value:
      pulumi:template: Python
  openstack:cloud:
    value: rxt
```

#### Step 4: Define an OpenStack Instance
Edit __main__.py and add the following code:
```python
"""

Copyright (C) 2024 DevX <victor.palma@rackspace.com>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <https://www.gnu.org/licenses/>.


A Python Pulumi program
"""

import pulumi_openstack as openstack
import pulumi

config = pulumi.Config()

demo_name = "openstack-demo"

server_flavor = config.require("bastion_server_flavor")
bastion_srv_name = "bastion01-" + demo_name

# Get External Network
ext_net_name = config.require("ext_net_name")
ext_net = openstack.networking.get_network(name=ext_net_name)

ssh_public_key = config.require("ssh_public_key")

# Create a key pair for SSH access
keypair = openstack.compute.Keypair(
    "keypair",
    name=demo_name + "-key",
    public_key=ssh_public_key,
)

# Create a Private network
private_network = openstack.networking.Network(
    demo_name + "-private-net",
    name=demo_name + "-private-net",
    admin_state_up=True,
)

# Create a subnet within the newly created private network
private_subnet = openstack.networking.Subnet(
    demo_name + "-private-subnet",
    name=demo_name + "-private-subnet",
    network_id=private_network.id,
    cidr="192.168.0.0/24",
    ip_version=4,
    dns_nameservers=["8.8.8.8"],
    #    gateway_ip=private_subnet_gateway,
)


# Create a router to connect the private network to the public network
router = openstack.networking.Router(
    demo_name + "-router",
    admin_state_up=True,
    external_network_id=ext_net.id,
)

router_interface = openstack.networking.RouterInterface(
    demo_name + "-router_interface",
    router_id=router.id,
    subnet_id=private_subnet.id,
)


# Create a security group allowing load balancer port
sec_group = openstack.networking.SecGroup(
    demo_name + "-sec_group",
    name=demo_name + "-sec_group",
    description="Security group for " + demo_name + " control plane",
    tags=[demo_name],
)

# Allow Load Balancer Ingress port 50000
openstack.networking.SecGroupRule(
    demo_name + "-allow_22_port",
    security_group_id=sec_group.id,
    direction="ingress",
    ethertype="IPv4",
    protocol="tcp",
    port_range_min=22,
    port_range_max=22,
    remote_ip_prefix="0.0.0.0/0",
)

bastion_port = openstack.networking.Port(
    bastion_srv_name,
    name=bastion_srv_name,
    network_id=private_network.id,
    fixed_ips=[{"subnet_id": private_subnet.id}],
    security_group_ids=[sec_group.id],
)

bastion = openstack.compute.Instance(
    bastion_srv_name,
    name=bastion_srv_name,
    flavor_name=server_flavor,
    image_id="727958e9-d037-45d1-9716-ea7ac322fe02",
    key_pair=keypair.name,
    networks=[{"port": bastion_port.id}],
    user_data =f"""#!/bin/bash
    apt-get update
    """,
)

bastion_fip = openstack.networking.FloatingIp(
        bastion_srv_name,
        pool=ext_net.name,
        port_id=bastion_port.id,
    )

pulumi.export("bastion_ip", bastion_fip.address)
```

#### Step 5: Deploy the Stack
Run the following commands:

Set your public key for the current `dev` stack:
```bash
pulumi config set ssh_public_key "$(cat ~/.ssh/id_ed25519.pub)"
pulumi config set ext_net_name PUBLICNET
pulumi config set  pulumi-openstack-flex-demo:bastion_server_flavor mo.0.2.16
```
!!! note

    Replace with your public ssh key, your External network Name and the flavor name if it's something different.

Preview the Changes:
```bash
pulumi preview
```
This command shows you what resources will be created and it should look something like following:
```bash
Enter your passphrase to unlock config/secrets
    (set PULUMI_CONFIG_PASSPHRASE or PULUMI_CONFIG_PASSPHRASE_FILE to remember):
Enter your passphrase to unlock config/secrets
Previewing update (dev):
     Type                                     Name                             Plan
 +   pulumi:pulumi:Stack                      getting-started-dev              create
 +   ├─ openstack:networking:SecGroupRule     openstack-demo-allow_22_port     create
 +   ├─ openstack:networking:Subnet           openstack-demo-private-subnet    create
 +   ├─ openstack:compute:Keypair             keypair                          create
 +   ├─ openstack:networking:Network          openstack-demo-private-net       create
 +   ├─ openstack:networking:SecGroup         openstack-demo-sec_group         create
 +   ├─ openstack:networking:Router           openstack-demo-router            create
 +   ├─ openstack:networking:Port             bastion01-openstack-demo         create
 +   ├─ openstack:networking:FloatingIp       bastion01-openstack-demo         create
 +   ├─ openstack:networking:RouterInterface  openstack-demo-router_interface  create
 +   └─ openstack:compute:Instance            bastion01-openstack-demo         create

Outputs:
    bastion_ip: output<string>

Resources:
    + 11 to create
```

Deploy the Stack:
```bash
pulumi up
```
Pulumi will display the proposed changes and prompt for confirmation. Type yes to proceed.


#### Step 6: Verify the Deployment
After deployment, Pulumi will output the instance_ip. You can SSH into your new instance:
```bash
ssh ubuntu@<instance_ip>
```
Replace <instance_ip> with the IP address provided in the output section:
```
Outputs:
    bastion_ip      : "XX.XXX.XXX.XXX".
```


## Conclusion
Congratulations! You've successfully deployed an OpenStack instance using Pulumi. This guide covered the basics of setting up Pulumi with OpenStack, but there's much more to explore. You can define more complex infrastructures, manage configurations, and leverage Pulumi's state management for robust deployments. The best part is that you can use the programing language of your choosing.

## Next Steps
  - Explore Pulumi Stacks: Learn how to manage different environments (e.g., dev, prod) using Pulumi stacks.
  - Pulumi Configuration: Use pulumi config to manage configuration variables securely.
  - CI/CD Integration: Integrate Pulumi deployments into your continuous integration and deployment pipelines.

## Resources
  - [Pulumi OpenStack Provider Documentation](https://www.pulumi.com/registry/packages/openstack/)
  - [OpenStack Documentation](https://docs.openstack.org/)
  - [Pulumi Official Website](https://www.pulumi.com/)

