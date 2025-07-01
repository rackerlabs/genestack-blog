---
date: 2024-06-18
title: Getting Started With Rackspace OpenStack Flex
authors:
  - jamesdenton
description: >
  Getting Started With Rackspace Rackspace OpenStack Flex
categories:
  - General
---

# Getting Started With Rackspace OpenStack Flex

We're now mid-way through 2024, and the Rackspace Public Cloud team has been hard at work building and delivering the successor to the Rackspace OpenStack Public Cloud (OSPC) first launched in 2012. As Limited Availability (LA) access to our latest multi-tenant OpenStack-based public cloud approaches and content begins to build up here on the blog, I thought it prudent to provide guidance on how one might onboard themselves and build resources on the platform.

<!-- more -->

Currently known as **OpenStack Flex**, this new multi-tenant cloud platform differs from our legacy OpenStack Public Cloud (OSPC) in many ways; most notably in its tenant-facing network architecture and adoption of KVM over Xenserver. Tenants have the flexibility to design their own network architecture within their virtual routing domain to create a more secure and performant environment. In addition, multiple storage backends will be available to handle workloads of all shapes and sizes. Have I mentioned **OpenStack** ***Flex*** is all about ***flexibility***?

## Basic requirements

Before a virtual machine instance can be created on the Rackspace OpenStack Flex Cloud, there are some prerequisites that must be provided by either the provider (Rackspace) and/or the tenant (user). A basic list includes:

- Cloud Credentials
- Image
- Flavor
- Network
- Router
- SSH Key
- Security Group(s)

## Configuring credentials

Rackspace OpenStack Flex is a true multi-tenant public cloud that leverages Rackspace Global Auth to provide access to cloud resources.

To begin creating cloud resources during the Limited Availability (LA) period, the OpenStack client will be required. On MacOS, installation of the client is easily accomplished using a Python virtual environment to isolate related files and libraries.

``` shell
$ mkdir ~/venvs; cd ~/venvs
$ python3 -m venv openstacksdk
$ source ~/venvs/openstacksdk/bin/activate
$ pip3 install openstacksdk
```

Once installed, create/update a `clouds.yaml` file containing your credentials that can be leveraged by the OpenStack client:

``` shell
$ mkdir ~/.config/openstack
$ cat <<EOF >>~/.config/openstack/clouds.yaml
cache:
  auth: true
clouds:
  rxt-flex-sjc3:
    auth:
      auth_url: https://<api endpoint>/v3
      project_name: <project name>
      project_domain_name: <project domain>
      username: <username>
      password: <token/password>
      user_domain_name: <user domain>
    region_name: SJC3
    interface: public
    identity_api_version: "3"
    insecure: false
EOF
```

See the configuration guide [here](https://docs.openstack.org/openstacksdk/latest/user/config/configuration.html) for more examples.

## Listing images

Rackspace provides unmodified cloud images as a convenience to users. To list these public images, use the `openstack image list --public` command as shown here:

``` shell
(openstacksdk) {8:44}~ ➭ openstack --os-cloud rxt-flex-sjc3 image list --public
+--------------------------------------+----------------------+--------+
| ID                                   | Name                 | Status |
+--------------------------------------+----------------------+--------+
| 3a5acf9b-af09-4b73-a7cb-182657823014 | AlmaLinux-8.9        | active |
| 1ca91d38-b416-4025-991e-b21ec7e69348 | AlmaLinux-9.4        | active |
| fe280193-475a-4c28-bbd7-b2d4cf8b12ac | ArchLinux-2024.05.15 | active |
| f8dcb046-a4bb-45cb-b831-3a9b00b7ff17 | CentOS-Stream-9      | active |
| 53131439-556c-42f3-b2fe-6d31f903daad | Debian-11            | active |
| b4d166b3-9764-4a23-ac8a-7dff9572a422 | Debian-12            | active |
| e3aa8daa-79c8-47a4-aa5d-d3770b3007f3 | Fedora-40            | active |
| 5e83637e-ca54-4c61-a115-87a76b68368e | Fedora-CoreOS-40     | active |
| 67e801cc-c87c-40a9-85cd-d026d137a2ad | FreeBSD-14.0-ufs     | active |
| a36a9a98-9e78-4c9b-98ab-2b7b40bd3e8b | FreeBSD-14.0-zfs     | active |
| f9bdde67-3cc0-4031-95b3-57e6028577b5 | Rocky-8.9            | active |
| 184443b3-3132-4eae-8ee8-4bfc9056591d | Rocky-9.4            | active |
| fcb944d4-35d4-4f3d-8d96-28c6a8b32c60 | Ubuntu-20.04         | active |
| 7ffb60c1-ca00-4a89-9ca9-a4aae1c0974a | Ubuntu-22.04         | active |
| f4874f53-3a3e-4529-871b-b8c0dd22d910 | Ubuntu-24.04         | active |
| b934dd0a-0db9-4ba5-b834-b8a1faa0cd61 | openSUSE-Leap-15     | active |
| e8a3258d-22d8-4617-bf0e-864b3a562f7a | openSUSE-Leap-15.6   | active |
+--------------------------------------+----------------------+--------+
```

!!! note "About those images"

    The listed images are provided free of charge and are subject to change. Some operating systems may require paid licenses. Contact your Rackspace support team for more information.

## Uploading images

Users are allowed to upload custom images, but be mindful that storing images in Flex may incur a nominal hourly fee. Check the fee schedule for more details.

## Flavors

Flavors are often referred to as "t-shirt sizes" for requested virtual machine resources. Rackspace has pre-defined flavors describing common virtual machine instance characteristics, including:

- vCPU Count
- RAM
- Disk

Using the `openstack flavor list` command, we can see the following pre-defined flavors are available for use:

``` shell
(openstacksdk) {11:36}~ ➭ openstack --os-cloud rxt-flex-sjc3 flavor list
+--------------------------------------+----------------+-------+------+-----------+-------+-----------+
| ID                                   | Name           |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+----------------+-------+------+-----------+-------+-----------+
| 232f467a-4dec-4100-8072-e046b758b5ad | m1.small       |  2048 |   20 |         0 |     2 | True      |
| 2debc6cb-d77f-4912-962b-c898f8db6fe1 | m1.extra_tiny  |   512 |    0 |         0 |     1 | True      |
| 574b6c02-0943-4a5a-8066-a13fa233743d | m1.extra_large | 16384 |  160 |        32 |     8 | True      |
| a2fe5ab2-7dac-4fae-afcf-fa5818bc1635 | m1.xlarge      | 16384 |  160 |         0 |     8 | True      |
| d47e32f7-165c-442f-89c5-cbc010279b13 | m1.medium      |  4096 |   40 |         8 |     4 | True      |
| df8a8c36-8095-4ca6-a64c-ca465c6e4b97 | m1.large       |  8192 |   80 |        16 |     6 | True      |
| f485b7c3-4efd-4c0d-b8b0-997db6bdbbce | m1.tiny        |  1024 |   10 |         0 |     1 | True      |
+--------------------------------------+----------------+-------+------+-----------+-------+-----------+
```

!!! note "Flavor sizing"

    Certain operating systems may have minimum requirements, so be sure to check with the vendor to see which flavor is most appropriate without overprovisioning. Rackspace provides a fee schedule for flavors and other resources that can be used to generate a total cost for the instance or virtual environment.

## Networks

In the Rackspace OpenStack Flex cloud, users within a project/tenant are now able to easily build a network architecture that consists of a virtual router at its core that is connected to an external *public* network and one or more internal *project/tenant* networks. Unlike the legacy OpenStack Public Cloud, instances utilize user-defined network address space and leverage SNAT and DNAT by default rather than be configured with public IPs directly.

Self-service networking is made possible by the use of a network overlay. In the case of OpenStack Flex, the overlay technology is known as GENEVE. GENEVE differs from VXLAN by providing a larger header that can be used to describe additional characteristics about the network at a lower level than what is available to tenants, but from a user perspective there is no practical difference. Overlay networks aren't always compatible with all network technologies, such as multicast, so feel free to reach out to the Rackspace OpenStack Flex support team to discuss additional network options or workarounds that might be available.

### Tenant network architecture

The core of tenant networking is a **virtual router**. It provides connectivity in and out of a tenant's virtual routing domain, and consists of a minimum of two interfaces: one external and one internal.

!!! info "Flex Virtual Router"

    ![Neutron Virtual Router](assets/images/2024-06-18/flex-virtual-router.png)

The **external** side of a virtual router connects to a single external provider network. External provider networks provide access to/from external networks, such as the Internet. This network is provided by Rackspace and is identified by the name **PUBLICNET**, as seen here:

``` shell
(openstacksdk) {8:46}~ ➭ openstack --os-cloud rxt-flex-sjc3 network list --external
+--------------------------------------+-----------+----------------------------------------------------------------------------+
| ID                                   | Name      | Subnets                                                                    |
+--------------------------------------+-----------+----------------------------------------------------------------------------+
| 723f8fa2-dbf7-4cec-8d5f-017e62c12f79 | PUBLICNET | 0e24addd-4b46-4dd8-9be3-08a351d724ad, 31bf7e05-be6e-4c5b-908d-abe47c80ba41 |
+--------------------------------------+-----------+----------------------------------------------------------------------------+
```

!!! info "A word about **PUBLICNET**"

    The **PUBLICNET** external provider network is a *shared* network that can only be leveraged by virtual routers. Attempts to place virtual machine instances directly onto **PUBLICNET** should fail.

The **internal** side of a virtual router connects to one of more networks defined within a project and created by a user of that project. Internal networks are traditionally connected to virtual machine instances. To list these internal networks, use the `openstack network list` command shown here:

``` shell
openstack --os-cloud rxt-flex-sjc3 network list --internal
```

### Creating a network and subnet

In OpenStack, a network is broken up into two distinct logical objects: the network itself that describes Layer 2 connectivity characteristics, such as physical network interface and segmentation ID (ie. VLAN ID), and a subnet that describes Layer 3 connectivity characteristics, including the corresponding CIDR/IP address space and other related attributes. A network can be associated with multiple subnets, but a subnet can only be associated with a single network.

New projects do not have networks created by default. To create a network, you'll need the following information:

- network name (required)

To create a subnet, you'll need or want the following:

- subnet name (required)
- network name or uuid (required)
- dns nameservers (optional)
- gateway ip (optional, default is 'auto')

The following example demonstrates creating a network and associated subnet:

- network name: tenant-net
- subnet name: tenant-subnet
- cidr: 192.168.100.0/24
- gateway: auto
- dns nameservers: 1.1.1.1, 1.0.0.1

``` shell
(openstacksdk) {10:47}~ ➭ openstack --os-cloud rxt-flex-sjc3 network create tenant-net

+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   | nova                                 |
| availability_zones        |                                      |
| created_at                | 2024-05-14T15:50:48Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 3673f3ad-21af-4445-85f9-b591d0c62362 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| l2_adjacency              | True                                 |
| mtu                       | 1442                                 |
| name                      | tenant-net                           |
| port_security_enabled     | True                                 |
| project_id                | 0136d6a661ef49b58235b09cb4af0ff7     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | 0136d6a661ef49b58235b09cb4af0ff7     |
| updated_at                | 2024-05-14T15:50:49Z                 |
+---------------------------+--------------------------------------+
```

``` shell
(openstacksdk) {10:50}~ ➭ openstack --os-cloud rxt-flex-sjc3 subnet create tenant-subnet \
--network tenant-net \
--subnet-range 192.168.100.0/24 \
--dns-nameserver 1.1.1.1 \
--dns-nameserver 1.0.0.1
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.100.2-192.168.100.254        |
| cidr                 | 192.168.100.0/24                     |
| created_at           | 2024-05-14T15:52:22Z                 |
| description          |                                      |
| dns_nameservers      | 1.0.0.1, 1.1.1.1                     |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 192.168.100.1                        |
| host_routes          |                                      |
| id                   | cec889d4-6e66-4eb7-b729-4aadba66668d |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | tenant-subnet                        |
| network_id           | 3673f3ad-21af-4445-85f9-b591d0c62362 |
| project_id           | 0136d6a661ef49b58235b09cb4af0ff7     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2024-05-14T15:52:22Z                 |
+----------------------+--------------------------------------+
```

Notice that the gateway ip is 192.168.100.1. This is the IP address that will be configured automatically on an *internal* interface of the virtual router, and will act as the gateway for the respective subnet and connected virtual machine instances. DHCP will provide other network characteristics to connected clients, including the specified DNS servers and any configured host routes.

### Creating a virtual router

Much like a physical router, the virtual router facilitates traffic between internal networks and subnets and also provides ingress and egress capabilities to VMs via its external interface.

By default, all virtual machine instances behind a virtual router can connect to remote networks using Source Network Address Translation, or Source NAT (SNAT). This method results in the virtual router performing IP masquerading using its external IP address which hides the real (or fixed) IP of the VM. For virtual machine instances that need direct *ingress* capabilities or static IP address assignments, a Floating IP can be configured that provides that 1:1 destination NAT translation (DNAT).

The create a virtual router, the following information is required:

- router name

To connect a virtual router to the network, you'll need the following:

- external network name or uuid
- one or more internal tenant networks (name or uuid)

A virtual router can be created and connected in multiple distinct steps, or as a collapsed operation, like so:

- router name: tenant-router
- external network: PUBLICNET

``` shell
(openstacksdk) {11:00}~ ➭ openstack --os-cloud rxt-flex-sjc3 router create tenant-router --external-gateway PUBLICNET

+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                                      |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                                                         |
| availability_zone_hints | nova                                                                                                                                                                                       |
| availability_zones      |                                                                                                                                                                                            |
| created_at              | 2024-05-14T16:01:03Z                                                                                                                                                                       |
| description             |                                                                                                                                                                                            |
| enable_ndp_proxy        | None                                                                                                                                                                                       |
| external_gateway_info   | {"network_id": "723f8fa2-dbf7-4cec-8d5f-017e62c12f79", "external_fixed_ips": [{"subnet_id": "0e24addd-4b46-4dd8-9be3-08a351d724ad", "ip_address": "63.131.145.194"}], "enable_snat": true} |
| flavor_id               | None                                                                                                                                                                                       |
| id                      | 5a79b5e2-2c7e-445b-a545-03f31a46d2c2                                                                                                                                                       |
| name                    | tenant-router                                                                                                                                                                              |
| project_id              | 0136d6a661ef49b58235b09cb4af0ff7                                                                                                                                                           |
| revision_number         | 3                                                                                                                                                                                          |
| routes                  |                                                                                                                                                                                            |
| status                  | ACTIVE                                                                                                                                                                                     |
| tags                    |                                                                                                                                                                                            |
| tenant_id               | 0136d6a661ef49b58235b09cb4af0ff7                                                                                                                                                           |
| updated_at              | 2024-05-14T16:01:04Z                                                                                                                                                                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

The preceding output demonstrates that SNAT is enabled by default, enabling connected virtual machine instances to reach outside resources with no further interaction. Clients connecting from behind the virtual router will appear as the external IP address of the router, 63.131.145.194.

To connect a tenant network to the router, you'll need the following:

- router name or uuid
- tenant subnet name or uuid

``` shell
openstack --os-cloud rxt-flex-sjc3 router add subnet tenant-router tenant-subnet
```

The operation does not provide any feedback to the user, but showing the properties of the virtual router will reveal its connected interfaces, like so:

``` shell
(openstacksdk) {11:03}~ ➭ openstack --os-cloud rxt-flex-sjc3 router show tenant-router

+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                                      |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                                                         |
| availability_zone_hints | nova                                                                                                                                                                                       |
| availability_zones      | nova                                                                                                                                                                                       |
| created_at              | 2024-05-14T16:01:03Z                                                                                                                                                                       |
| description             |                                                                                                                                                                                            |
| enable_ndp_proxy        | None                                                                                                                                                                                       |
| external_gateway_info   | {"network_id": "723f8fa2-dbf7-4cec-8d5f-017e62c12f79", "external_fixed_ips": [{"subnet_id": "0e24addd-4b46-4dd8-9be3-08a351d724ad", "ip_address": "63.131.145.194"}], "enable_snat": true} |
| flavor_id               | None                                                                                                                                                                                       |
| id                      | 5a79b5e2-2c7e-445b-a545-03f31a46d2c2                                                                                                                                                       |
| interfaces_info         | [{"port_id": "cc370deb-10f6-4b5c-a718-b040254ddc1d", "ip_address": "192.168.100.1", "subnet_id": "cec889d4-6e66-4eb7-b729-4aadba66668d"}]                                                  |
| name                    | tenant-router                                                                                                                                                                              |
| project_id              | 0136d6a661ef49b58235b09cb4af0ff7                                                                                                                                                           |
| revision_number         | 4                                                                                                                                                                                          |
| routes                  |                                                                                                                                                                                            |
| status                  | ACTIVE                                                                                                                                                                                     |
| tags                    |                                                                                                                                                                                            |
| tenant_id               | 0136d6a661ef49b58235b09cb4af0ff7                                                                                                                                                           |
| updated_at              | 2024-05-14T16:03:52Z                                                                                                                                                                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

The `interfaces` field shows a single connected subnet with a configured IP address of 192.168.100.1 - otherwise known as the gateway IP of the `tenant-net` network and subnet.

## Creating an SSH key

When using cloud images from major operating system providers such a Canonical and Red Hat, including Ubuntu Server and CentOS Stream, you'll need an SSH key that can be injected into the virtual machine instance at boot in order to authenticate and log in.

Creating an SSH key can be accomplished in one of two ways:

- using an existing public/private key pair
- generating a new public/private key pair

### Generating a new keypair

Generating a new keypair is quick and painless using the `openstack keypair create` command, like so:

- keypair name: tenant-key

``` shell
(openstacksdk) {11:09}~ ➭ openstack --os-cloud rxt-flex-sjc3 keypair create tenant-key

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZWQyNTUx
OQAAACCD8JtLVD2v7woDUl3iYjwLdwbNkN2ypNdf0W3jzcbzsQAAAIjuqj1J7qo9SQAAAAtzc2gt
ZWQyNTUxOQAAACCD8JtLVD2v7woDUl3iYjwLdwbNkN2ypNdf0W3jzcbzsQAAAEBU7iX9UWtGGb6/
fXqXYrTMkuIwZtSz8lL7g+fokxXyjoPwm0tUPa/vCgNSXeJiPAt3Bs2Q3bKk11/RbePNxvOxAAAA
AAECAwQF
-----END OPENSSH PRIVATE KEY-----
```

The client will output the contents of the private key to stdout while storing the public key in the database:

``` shell
(openstacksdk) {11:11}~ ➭ openstack --os-cloud rxt-flex-sjc3 keypair show tenant-key

+-------------+------------------------------------------------------------------+
| Field       | Value                                                            |
+-------------+------------------------------------------------------------------+
| created_at  | 2024-05-14T16:11:29.000000                                       |
| fingerprint | d4:86:70:cb:50:b2:a7:dc:61:7d:8c:52:15:5e:2e:eb                  |
| id          | tenant-key                                                       |
| is_deleted  | False                                                            |
| name        | tenant-key                                                       |
| private_key | None                                                             |
| type        | ssh                                                              |
| user_id     | dc7d79d748c69f8a7027e987350d0236bf10a070823a645fb3f22e2aa16c76d2 |
+-------------+------------------------------------------------------------------+
```

!!! tip "Watch that key!"

    Be sure ***not*** to lose or share the private key, as it may result in a compromise of associated virtual machine instances. To save the private key, create a file at `~/.ssh/tenant-key` with the contents and set permissions accordingly using `chmod`:

    `$ chmod 600 ~/.ssh/tenant-key`

### Creating a keypair using an existing public key

Should you want to leverage an existing SSH keypair on your local machine, use the `openstack keypair create` command with the `--public-key` option, like so:

- keypair name: existing-tenant-key
- public key path: `~/.ssh/id_rsa.pub`

``` shell
(openstacksdk) {11:21}~ ➭ openstack --os-cloud rxt-flex-sjc3 keypair create --public-key ~/.ssh/id_rsa.pub existing-tenant-key
+-------------+------------------------------------------------------------------+
| Field       | Value                                                            |
+-------------+------------------------------------------------------------------+
| created_at  | None                                                             |
| fingerprint | a5:45:85:b6:e6:e6:84:0e:05:67:08:82:07:02:a4:27                  |
| id          | existing-tenant-key                                              |
| is_deleted  | None                                                             |
| name        | existing-tenant-key                                              |
| type        | ssh                                                              |
| user_id     | dc7d79d748c69f8a7027e987350d0236bf10a070823a645fb3f22e2aa16c76d2 |
+-------------+------------------------------------------------------------------+
```

Keypairs can be listed using the `openstack keypair list` command, as shown here:

``` shell
(openstacksdk) {11:29}~ ➭ openstack --os-cloud rxt-flex-sjc3 keypair list
+---------------------+-------------------------------------------------+------+
| Name                | Fingerprint                                     | Type |
+---------------------+-------------------------------------------------+------+
| existing-tenant-key | a5:45:85:b6:e6:e6:84:0e:05:67:08:82:07:02:a4:27 | ssh  |
| tenant-key          | d4:86:70:cb:50:b2:a7:dc:61:7d:8c:52:15:5e:2e:eb | ssh  |
+---------------------+-------------------------------------------------+------+
```

## Security Groups

**Security groups** are sets of IP filter rules that are applied to all servers that are members of the group. **Security group rules** define networking access to associated virtual servers, and can reference source IP addresses or even other groups. Group rules are project-specific, and project members can edit the rules for their groups and add new rule sets depending on role. OpenStack operates on a **default deny** basis, meaning no traffic *in* or *out* is allowed without an explicit rule allowing that traffic.

All projects have a `default` security group which is applied to any port that has no other security group defined. Unless you change the default, this security group denies all incoming traffic and allows only outgoing traffic from the instance. When troubleshooting connectivity to a newly-launched VM, it is important to review the security group rules to make sure access is allowed.

!!! note "Security group limits"

    All projects within the Rackspace OpenStack Flex cloud have a quota that sets a maximum limit to the number of security groups and rules that can be created within a project. At this time, a project is limited to ten (10) security groups and a total of one-hundred (100) security group rules across those groups. To increase your quota, please reach out to the Rackspace OpenStack Flex support team.

To create a new security group and associated rule(s), the following is required:

- security group name
- traffic direction (ingress or egress)
- source address (cidr) -or-
- source group (name or uuid)
- protocol (tcp, udp, icmp, etc.)
- destination port

To create a new group, perform the following:

- security group name: tenant-secgroup

``` shell
(openstacksdk) {12:49}~ ➭ openstack --os-cloud rxt-flex-sjc3 security group create tenant-secgroup
+-----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field           | Value                                                                                                                                                                            |
+-----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at      | 2024-05-14T17:49:49Z                                                                                                                                                             |
| description     | tenant-secgroup                                                                                                                                                                  |
| id              | 52b8fe21-a06f-424b-aa12-dba39ca371c6                                                                                                                                             |
| name            | tenant-secgroup                                                                                                                                                                  |
| project_id      | 0136d6a661ef49b58235b09cb4af0ff7                                                                                                                                                 |
| revision_number | 1                                                                                                                                                                                |
| rules           | created_at='2024-05-14T17:49:49Z', direction='egress', ethertype='IPv6', id='6b6ff426-bf20-4091-89f6-12822824fa92', standard_attr_id='297496', updated_at='2024-05-14T17:49:49Z' |
|                 | created_at='2024-05-14T17:49:49Z', direction='egress', ethertype='IPv4', id='f2e4bed0-d885-45da-bc3a-5ecea52ef7e2', standard_attr_id='297491', updated_at='2024-05-14T17:49:49Z' |
| shared          | False                                                                                                                                                                            |
| stateful        | True                                                                                                                                                                             |
| tags            | []                                                                                                                                                                               |
| updated_at      | 2024-05-14T17:49:49Z                                                                                                                                                             |
+-----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

The creation of the security group has triggered the creation of two *egress* rules - one for IPv4 and the other for IPv6. A quick look at the IPv4 rule shows the following:

``` shell
(openstacksdk) {12:49}~ ➭ openstack --os-cloud rxt-flex-sjc3 security group rule show f2e4bed0-d885-45da-bc3a-5ecea52ef7e2
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| created_at              | 2024-05-14T17:49:49Z                 |
| description             | None                                 |
| direction               | egress                               |
| ether_type              | IPv4                                 |
| id                      | f2e4bed0-d885-45da-bc3a-5ecea52ef7e2 |
| name                    | None                                 |
| normalized_cidr         | None                                 |
| port_range_max          | None                                 |
| port_range_min          | None                                 |
| project_id              | 0136d6a661ef49b58235b09cb4af0ff7     |
| protocol                | None                                 |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 52b8fe21-a06f-424b-aa12-dba39ca371c6 |
| tags                    | []                                   |
| updated_at              | 2024-05-14T17:49:49Z                 |
+-------------------------+--------------------------------------+
```

The rule states that *egress* traffic from the instance to 0.0.0.0/0 (all hosts) is allowed.

To create a new rule allowing *ingress* SSH access to instances, perform the following:

- security group name: tenant-secgroup
- traffic direction: ingress
- source address: 0.0.0.0/0
- protocol: tcp
- destination port: 22

``` shell
(openstacksdk) {12:54}~ ➭ openstack --os-cloud rxt-flex-sjc3 security group rule create tenant-secgroup \
--protocol tcp \
--ingress \
--remote-ip 0.0.0.0/0 \
--dst-port 22
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| created_at              | 2024-05-14T17:54:52Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | fbaa4bab-f0d3-41d1-a0c6-909b366c0373 |
| name                    | None                                 |
| normalized_cidr         | 0.0.0.0/0                            |
| port_range_max          | 22                                   |
| port_range_min          | 22                                   |
| project_id              | 0136d6a661ef49b58235b09cb4af0ff7     |
| protocol                | tcp                                  |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 52b8fe21-a06f-424b-aa12-dba39ca371c6 |
| tags                    | []                                   |
| updated_at              | 2024-05-14T17:54:52Z                 |
+-------------------------+--------------------------------------+
```
!!! note "A note about client IPs"

    Defining 0.0.0.0/0 as the client or remote IP is insecure, as it allows any client IP address the ability to SSH to the instance. That's fine for the purpose of this demonstration, but should be tightened down accordingly for your environment.

## Launching a VM

Now that the foundation is in place, a virtual machine can be launched on the platform. The following information will be required:

- instance name
- network name or uuid
- image name or uuid
- keypair name or uuid
- flavor name or uuid
- security group name or uuid

To launch an instance with the following characteristics, use the `openstack server create` command as shown:

- instance name: Server01
- network name: tenant-net
- image name: Ubuntu-22.04
- keypair name: tenant-key
- flavor name: m1.small
- security group name: tenant-secgroup

``` shell
(openstacksdk) {12:56}~ ➭ openstack --os-cloud rxt-flex-sjc3 server create Server01 \
--network tenant-net \
--image Ubuntu-22.04 \
--key-name tenant-key \
--flavor m1.small \
--security-group tenant-secgroup
+--------------------------------------+------------------------------------------------------------------+
| Field                                | Value                                                            |
+--------------------------------------+------------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                           |
| OS-EXT-AZ:availability_zone          |                                                                  |
| OS-EXT-STS:power_state               | NOSTATE                                                          |
| OS-EXT-STS:task_state                | scheduling                                                       |
| OS-EXT-STS:vm_state                  | building                                                         |
| OS-SRV-USG:launched_at               | None                                                             |
| OS-SRV-USG:terminated_at             | None                                                             |
| accessIPv4                           |                                                                  |
| accessIPv6                           |                                                                  |
| addresses                            |                                                                  |
| adminPass                            | vGV2wtBC5ePg                                                     |
| config_drive                         |                                                                  |
| created                              | 2024-05-14T18:00:50Z                                             |
| flavor                               | m1.small (232f467a-4dec-4100-8072-e046b758b5ad)                  |
| hostId                               |                                                                  |
| id                                   | ded52112-1961-40fe-bdd8-91c42897fd81                             |
| image                                | Ubuntu-22.04 (c57007d9-096e-403f-9479-c91d86438d83)              |
| key_name                             | tenant-key                                                       |
| name                                 | Server01                                                         |
| os-extended-volumes:volumes_attached | []                                                               |
| progress                             | 0                                                                |
| project_id                           | 0136d6a661ef49b58235b09cb4af0ff7                                 |
| properties                           |                                                                  |
| security_groups                      | name='52b8fe21-a06f-424b-aa12-dba39ca371c6'                      |
| status                               | BUILD                                                            |
| updated                              | 2024-05-14T18:00:50Z                                             |
| user_id                              | dc7d79d748c69f8a7027e987350d0236bf10a070823a645fb3f22e2aa16c76d2 |
+--------------------------------------+------------------------------------------------------------------+
```

Using the `openstack server list` command, you can observe the state of all virtual machine instances within the project:

``` shell
(openstacksdk) {11:40}~ ➭ openstack --os-cloud rxt-flex-sjc3 server list
+--------------------------------------+----------+--------+----------------------------+--------------+----------+
| ID                                   | Name     | Status | Networks                   | Image        | Flavor   |
+--------------------------------------+----------+--------+----------------------------+--------------+----------+
| ded52112-1961-40fe-bdd8-91c42897fd81 | Server01 | ACTIVE | tenant-net=192.168.100.254 | Ubuntu-22.04 | m1.small |
+--------------------------------------+----------+--------+----------------------------+--------------+----------+
```

!!! tip "IP address assignment"

    IP address assignment within a subnet's allocation pool is random, but can be supplied as a hint during instance creation to ensure a static assignment. Reference the OpenStack client `help` command for more details.

### Connecting to the virtual machine instance

At long last, the instance is `ACTIVE` and ready for connections. Right? Well, yes and no. Because the instance's IP is not reachable from the Internet ([See RFC1918](https://datatracker.ietf.org/doc/html/rfc1918)) we must first associate a floating IP. As mentioned in an earlier section of this walkthrough, the **PUBLICNET** network attached to the external side of our virtual router will provide the much-needed **Floating IP** address space needed to allow direct *ingress* connections to the instance.

!!! note "Floating IP Two-Step"

    Floating IP *creation* and *association* are two distinct steps in the API. In addition, a floating IP is loosely coupled to an instance via the instance's network port rather than the instance itself. Any association must reference the port ID and not the instance ID.

To create a Floating IP, you'll need the following information:

- Floating IP Network

To associate the Floating IP with an instance and/or port, you'll need the following:

- Floating IP ID
- Port ID

The create a Floating IP, perform the following:

``` shell
(openstacksdk) {14:03}~ ➭ openstack --os-cloud rxt-flex-sjc3 floating ip create PUBLICNET

+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| created_at          | 2024-05-14T19:03:49Z                 |
| description         |                                      |
| dns_domain          |                                      |
| dns_name            |                                      |
| fixed_ip_address    | None                                 |
| floating_ip_address | 63.131.145.210                       |
| floating_network_id | 723f8fa2-dbf7-4cec-8d5f-017e62c12f79 |
| id                  | 2be6834d-2a74-484b-818f-65484247d717 |
| name                | 63.131.145.210                       |
| port_details        | None                                 |
| port_id             | None                                 |
| project_id          | 0136d6a661ef49b58235b09cb4af0ff7     |
| qos_policy_id       | None                                 |
| revision_number     | 0                                    |
| router_id           | None                                 |
| status              | DOWN                                 |
| subnet_id           | None                                 |
| tags                | []                                   |
| updated_at          | 2024-05-14T19:03:49Z                 |
+---------------------+--------------------------------------+
```

!!! note "About that Floating IP"

    A floating IP is allocated from an external provider network and can be mapped to one port at a time. It gets the designation of 'floating' by the ability to *disassociate* from one port and *associate* with another without needing to make any upstream network changes. Thus, the IP "floats" between instances. This 1:1 mapping of Floating IP -> Port can be thought of as a Destination NAT or Static NAT.

To associate the floating IP, you'll first need to find the ID of the instance's port and then perform the association. Perform the following to find the ports of an instance and make the association:

``` shell
(openstacksdk) {14:06}~ ➭ openstack --os-cloud rxt-flex-sjc3 port list --device-id ded52112-1961-40fe-bdd8-91c42897fd81

+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                             | Status |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
| 951055eb-a62f-4cb7-a5fd-eaae9b138c95 |      | fa:16:3e:8f:51:c0 | ip_address='192.168.100.254', subnet_id='cec889d4-6e66-4eb7-b729-4aadba66668d' | ACTIVE |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+

(openstacksdk) {14:07}~ ➭ openstack --os-cloud rxt-flex-sjc3 floating ip set 2be6834d-2a74-484b-818f-65484247d717 --port 951055eb-a62f-4cb7-a5fd-eaae9b138c95
```

The association operation does not provide any feedback to the user, but listing the floating IPs will demonstrate the association, like so:

``` shell
(openstacksdk) {14:08}~ ➭ openstack --os-cloud rxt-flex-sjc3 floating ip list

+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| ID                                   | Floating IP Address | Fixed IP Address | Port                                 | Floating Network                     | Project                          |
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
| 2be6834d-2a74-484b-818f-65484247d717 | 63.131.145.210      | 192.168.100.254  | 951055eb-a62f-4cb7-a5fd-eaae9b138c95 | 723f8fa2-dbf7-4cec-8d5f-017e62c12f79 | 0136d6a661ef49b58235b09cb4af0ff7 |
+--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
```

Armed with the floating IP address, you can now attempt a connection to the virtual machine instance:

``` shell
(openstacksdk) {14:10}~ ➭ ssh -i ~/.ssh/tenant-key ubuntu@63.131.145.210
Warning: Permanently added '63.131.145.210' (RSA) to the list of known hosts.
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-102-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System information as of Tue May 14 19:10:26 UTC 2024

  System load:  0.0               Processes:               114
  Usage of /:   7.3% of 19.20GB   Users logged in:         0
  Memory usage: 10%               IPv4 address for enp3s0: 192.168.100.254
  Swap usage:   0%

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@server01:~$
```

Some quick validation of the VM shows it's running Ubuntu 22.04 and has the defined flavor characteristics:

``` shell
ubuntu@server01:~$ curl https://icanhazip.com
63.131.145.210
ubuntu@server01:~$ uname -a
Linux server01 5.15.0-102-generic #112-Ubuntu SMP Tue Mar 5 16:50:32 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux

ubuntu@server01:~$ lscpu
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         40 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  2
  On-line CPU(s) list:   0,1
Vendor ID:               AuthenticAMD
  Model name:            AMD EPYC Processor
    CPU family:          23
    Model:               1
    Thread(s) per core:  1
    Core(s) per socket:  1
    Socket(s):           2
    Stepping:            2
    BogoMIPS:            5888.93

ubuntu@server01:~$ free --mega
               total        used        free      shared  buff/cache   available
Mem:            2052         184        1472           4         395        1703
Swap:              0           0           0

ubuntu@server01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           196M  1.1M  195M   1% /run
/dev/sda1        20G  1.5G   18G   8% /
tmpfs           979M     0  979M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/sda15      105M  6.1M   99M   6% /boot/efi
tmpfs           196M  4.0K  196M   1% /run/user/1000
```

## Conclusion

It may seem like a lot of steps to onboard a VM, but 90% of what has been described and built here is a **One Time Thing™** that sets the foundation for subsequent launches. In the future, we'll demonstrate how to leverage tools like Ansible and Terraform to speed up these same operations in an idempotent and repeatable fashion.

Cheers!
