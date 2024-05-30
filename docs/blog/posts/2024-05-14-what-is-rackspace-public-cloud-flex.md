---
date: 2024-05-14
title: What is Rackspace Public Cloud (Flex)
authors:
  - jamesdenton
description: >
  What is Rackspace Public Cloud (Flex)
categories:
  - General
---

# What is Rackspace Public Cloud (Flex)?

In 2006, before 'The Cloud' became a ubiquitous term, Rackspace launched one of the first utility-based computing services known as Mosso (later Rackspace Cloud). Move ahead to 2010, and you'll find Rackspace partnering with NASA to deliver the first release of OpenStack - a fully open-source and open standard cloud computing platform. In 2012, Rackspace launched the Rackspace Public Cloud based on OpenStack, and shortly thereafter began delivering private (hosted) clouds based on OpenStack. To say OpenStack runs through our veins is an understatement.
<!-- more -->

In 2024, the Rackspace Cloud team has been hard at work developing the next-generation multi-tenant Rackspace Public Cloud based on OpenStack, and I'm happy to share that the doors will be open in limited availability in coming weeks. In this blog, I'd like to take you through a journey into the architecture of Flex - the next-generation public cloud by Rackspace.

## GeneStack: The Foundation of Rackspace OpenStack Flex

GeneStack (Genesis+OpenStack) is the code name for our latest OpenStack implementation based on OpenStack-Helm. By leveraging Kubernetes, GeneStack offers flexibility and scalability of cloud infrastructure that wasn't easily possible using yesterday's tooling. Kubernetes also offers operational benefits and visibility that simplify cloud management. Day-to-day, this architecture shift is expected to be transparent to the user, but offers a serious upside when it comes to upgrades and other maintenance of the cloud infrastructure.

### Networking

The Rackspace OpenStack Flex cloud differs from the legacy OpenStack Public Cloud (OSPC) in many ways, but especially when it comes to networks. Rackspace has adopted a leaf/spine architecture along with BGP eVPN to provide scale that could not be achieved with the traditional core-aggr-tor design. Rather than virtual machine instances being connected to a front-end PUBLICNET and back-end SERVICENET network by default, users within a project are now able to easily define a network architecture that consists of a virtual router connected to an external 'public' provider network and one or more inside 'tenant' networks. No longer will instances be configured with a public IP directly, rather, they will utilize a user-defined network address space and leverage source NAT (SNAT) and destination NAT (DNAT) where required. In OpenStack parlance, this is NAT functionality is known as Floating IP - or Elastic IP if you're coming from AWS.

![Virtual Route Domain](assets/images/2024-05-14/virtual_route_domain2.png)

Self-service networking is made possible by the use of a network overlay. OpenStack Flex uses GENEVE as the overlay technology with Open Virtual Network (OVN) as the underlying network plugin. GENEVE differs from VXLAN by providing a larger header that can be used to describe additional characteristics about the network at a lower level than what is available to tenants. The use of an overlay allows for network scaling with limited impact to performance. While there could be limitations for certain traffic types, such as multicast, most network traffic is supported. Feel free to reach out to the Rackspace OpenStack Flex support team to discuss additional options that might be available.

### Compute

The first regions of Flex will see 15th and 16th generation Dell enterprise servers enter the ring equipped with AMD EPYC processors. In a substantial shift from OSPC and a nod to current practices, Rackspace has adopted KVM as the hypervisor for Rackspace OpenStack Flex. Rackspace's Enterprise OpenStack product team has well over a decade of experience running large KVM-based OpenStack private clouds, and while XenServer has served us well for nearly 15 years, it's time to say goodbye. Workloads in OSPC can be migrated to Flex, but the details are still being worked out. If you're interested in migrating, feel free to reach out to your Rackspace support team for assistance.

Users will be allowed to consume upstream cloud images from major providers such as Microsoft, Canonical, SUSE, Red Hat, and others. In addition, users can upload custom images for a nominal monthly fee.

### Storage

Rackspace expects to offer multiple commodity and Enterprise-grade storage solutions to meet the needs of all workloads. Both block and object storage will be available equipped with high-performant SSD and NVMe options. Customers can launch volume-backed instances, perform snapshots of instances and volumes, and leverage both Swift and S3-compatible APIs for object storage. Multiple block storage backends offer a choice of 'nines' based on your desired level of risk and price point.

## Join us

Rackspace OpenStack Flex is a natural successor to OSPC, and the start of a new 'flexible' foundation for upcoming Rackspace technologies and offerings. If you're interested in testing driving the multi-tenant Rackspace OpenStack Flex cloud or are looking for a dedicated private cloud ala Rackspace OpenStack Enterprise, give us a call!
