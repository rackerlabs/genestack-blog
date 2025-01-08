---
date: 2024-12-20
title: Using ansible to create resources on flex cloud
authors:
  - puni4220
description: >
  Using ansible to create resources on flex cloud
categories:
  - ansible
  - openstack
---

# Using ansible to create resources on flex cloud

Ansible has a wide range of modules available to create and manage resources like openstack flavors, images, keypairs, networks, routers
among others on the flex cloud. These modules are available in the **Openstack.Cloud** ansible collection. In this post we will discuss
creating resources on flex cloud using ansible modules.

<!-- more -->

# Pre-requisites

  There are certain pre-requisites which are necessary to create resources with ansible:

* A python virtual environment (though not strictly necessary but recommended) with ansible installed. The install documention
  [bootstrap script](https://docs.rackspacecloud.com/genestack-getting-started/) takes care of creating the python venv and
  installing the required versions of **ansible** and **openstack.cloud** ansible collection. The versions which are installed
  by bootstrap script in this example:
  ```shell
  ansible                   8.5.0
  ansible-core              2.15.13

  # /root/.venvs/genestack/lib/python3.10/site-packages/ansible_collections
  Collection      Version
  --------------- -------
  openstack.cloud 2.1.0  
  ```

* Once ansible and openstack.cloud collection is installed we will need to install the **openstacksdk**. This is required by modules
  in openstack.cloud ansible collection to create resources on flex cloud; this can be simply installed with pip:
  ```shell
  (genestack) root@bastn:~# pip install openstacksdk

  (genestack) root@bastn:~# pip show openstacksdk
  Name: openstacksdk
  Version: 4.1.0
  Summary: An SDK for building applications to work with OpenStack
  Home-page: https://docs.openstack.org/openstacksdk/
  Author: OpenStack
  Author-email: openstack-discuss@lists.openstack.org
  License: UNKNOWN
  Location: /root/.venvs/genestack/lib/python3.10/site-packages
  Requires: cryptography, decorator, dogpile.cache, iso8601, jmespath, jsonpatch, keystoneauth1, netifaces, os-service-types, pbr, 
  platformdirs, PyYAML, requestsexceptions
  Required-by: os-client-config, osc-lib, python-openstackclient
  ```

# Creating a project, user and a flavor in flex cloud

  The purpose of this step is create a project and a user in flex cloud; We will be creating resources with the credentials of this
  user in flex cloud. Users in member role can't create flavors. It should be noted that this is an admin only action and will 
  require admin credentials:
  ```shell
  (genestack) root@bastn:~# openstack project create --description 'test project 2' project2
  +-------------+----------------------------------+
  | Field       | Value                            |
  +-------------+----------------------------------+
  | description | test project 2                   |
  | domain_id   | default                          |
  | enabled     | True                             |
  | id          | 292e84ccc2f84912955cd2c5e473a961 |
  | is_domain   | False                            |
  | name        | project2                         |
  | options     | {}                               |
  | parent_id   | default                          |
  | tags        | []                               |
  +-------------+----------------------------------+

  (genestack) root@bastn:~# openstack user create user2 --password test123456 --project project2
  +---------------------+----------------------------------+
  | Field               | Value                            |
  +---------------------+----------------------------------+
  | default_project_id  | 292e84ccc2f84912955cd2c5e473a961 |
  | domain_id           | default                          |
  | enabled             | True                             |
  | id                  | b1a4e5cca1d9460d8229d03d77b1b20d |
  | name                | user2                            |
  | options             | {}                               |
  | password_expires_at | None                             |
  +---------------------+----------------------------------+
  (genestack) root@bastn:~# 

  (genestack) root@bastn:~# openstack role add --user user2 --project project2 member
  (genestack) root@bastn:~#

  (genestack) root@bastn:~# openstack flavor create m1.medium1 --vcpus 1 --ram 2048 --disk 10 
  +----------------------------+--------------------------------------+
  | Field                      | Value                                |
  +----------------------------+--------------------------------------+
  | OS-FLV-DISABLED:disabled   | False                                |
  | OS-FLV-EXT-DATA:ephemeral  | 0                                    |
  | description                | None                                 |
  | disk                       | 10                                   |
  | id                         | 681fc077-af91-4a7d-affb-a3bbefac4ef4 |
  | name                       | m1.medium1                           |
  | os-flavor-access:is_public | True                                 |
  | properties                 |                                      |
  | ram                        | 2048                                 |
  | rxtx_factor                | 1.0                                  |
  | swap                       | 0                                    |
  | vcpus                      | 1                                    |
  +----------------------------+--------------------------------------+
  ```

# Export the required auth parameters for openstacksdk as environment variables

  openstacksdk library requires auth parameters like username, password, project and other details like keystone auth url to
  create the resources with ansible. These parameters can be specified either at the level of an individual task or as 
  environment variables. In this example we will export the required variables as environment variables by creating a file:

  ```shell
  (genestack) root@bastn:~# cat user2rc 
  export OS_ENDPOINT_TYPE=public
  export OS_REGION_NAME=RegionOne
  export OS_INTERFACE=public
  export OS_AUTH_URL=https://keystone.cluster.local/v3
  export OS_PROJECT_DOMAIN_NAME=default
  export OS_DEFAULT_DOMAIN=default
  export OS_USERNAME=user2
  export OS_USER_DOMAIN_NAME=default
  export OS_PROJECT_NAME=project2
  export OS_PASSWORD=test123456
  export OS_IDENTITY_API_VERSION=3
  ```
  notice that these are credentials for the user we created along with other variables like domain name for user and project
  and the keystone auth url (public endpoint). This file will need to be sourced to create the resources with ansible

# Creating the required file for ansible variables

  Before we can create the resources with ansible we will need to create a yaml file for the variables which will be used in 
  ansible playbooks. These are comman variables like instance name, subnet name, image name and net name among others. It should
  be noted that these are not strictly required but it is recommend to use variables instead of static names to re-use playbooks

  ```shell
  (genestack) root@bastn:~# cat vars.yml 
  ssh_key_name: tstk1
  net_name: net8
  subnet_name: subnet8
  subnet_cidr: 172.16.80.0/24
  security_group_name: secgroup1
  router_name: router2
  external_net_name: extn
  port_name: port1
  image_name: ubuntu20-test1
  qcow2_name: focal-server-cloudimg-amd64.img 
  volume_name: ubuntu20-volume1
  volume_size: 5 # must be according to the virtual size of the image
  flavor_name: m1.medium
  instance_name: test12345
  ```
  notice the **volume_size** variable in this example. This must be set according to the virtual size of the qcow2 image i.e
  if you are trying to create a bootable volume then the volume size must be either equal to or greater than the size of qcow2
  It is possible to capture the virtual size of the image by running a command like:

  ```shell
  (genestack) root@bastn:~# qemu-img info /tmp/focal-server-cloudimg-amd64.img
  ```
  the virtual size of the image in this example is 2G and therefore the volume size of 5G is sufficient

  # Creating the playbook 

  There are several modules to create resources with ansible in flex cloud. The purpose of this example is to demonstrate creating
  resources like ssh keys, network, security group, subnet, router, network port, image, bootable volume and an instance

  ```shell
  (genestack) root@bastn:~# cat test1.yml
  ---
  - name: Playbook to create infra on OpenStack flex cloud with ansible
    hosts: localhost
    gather_facts: false
    vars_files:
      - vars.yml
    tasks:
      - name: check if the keypair by the name {{ ssh_key_name }} already exists
        ansible.builtin.stat:
          path: "{{ lookup('env', 'HOME') }}/{{ ssh_key_name }}"
        register: _ssh_key

      - name: if required create a keypair for the instances
        community.crypto.openssh_keypair:
          path: "{{ lookup('env', 'HOME') }}/{{ ssh_key_name }}"
          type: rsa
          size: 2048
        when: not _ssh_key.stat.exists

      - name: check if an instance already exists with name {{ instance_name }} in the openstack project {{ lookup('env',   'OS_PROJECT_NAME') }}
        block:
        - name: Obtain the details of the existing instances
          openstack.cloud.server_info:
            name: "{{ instance_name | default('test') }}"
          register: _instance_details

        - name: Fail if there is an existing by the name {{ instance_name }} in the same project
          ansible.builtin.fail:
            msg: There is an existing instance with the same name in the current project
          when: instance_name in _instance_details.servers | selectattr('name', 'defined') | map(attribute='name')

      - name: create a keypair in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.keypair:
          name: "{{ ssh_key_name }}"
          state: present
          public_key_file: "{{ lookup('env', 'HOME')}}/{{ ssh_key_name }}.pub"

      - name: create the network {{ net_name }} in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.network:
          name: "{{ net_name }}"
          state: present
        register: _network_details

      - name: create the subnet {{ subnet_name }} for network {{ net_name }} in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.subnet:
          name: "{{ subnet_name }}"
          state: present
          network: "{{ _network_details.id }}"
          cidr: "{{ subnet_cidr }}"
        register: _subnet_details

      - name: create the security group {{ security_group_name }} in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.security_group:
          name: "{{ security_group_name }}"
          state: present
          security_group_rules:
            - ether_type: IPv6
              direction: egress
            - ether_type: IPv4
              direction: egress
            - ether_type: IPv4
              direction: ingress
              port_range_max: 22
              port_range_min: 22
              protocol: tcp
              remote_ip_prefix: 0.0.0.0/0
            - ether_type: IPv4
              direction: ingress
              protocol: icmp
            - ether_type: IPv4
              port_range_max: 80
              port_range_min: 80
              protocol: tcp
              remote_ip_prefix: 0.0.0.0/0

      - name: create the router {{ router_name }} in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.router:
          name: "{{ router_name }}"
          state: present
          network: "{{ external_net_name }}"
          interfaces:
            - "{{ _subnet_details.id }}"

      - name: create the port {{ port_name }} in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.port:
          name: "{{ port_name }}"
          network: "{{ _network_details.id }}"
          state: present
          security_groups: "{{ security_group_name }}"
        register: _port_details

      - name: if required fetch ubuntu20 cloud image
        get_url:
          url: https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
          dest: /tmp/{{ qcow2_name }}

      - name: create the image {{ image_name }} in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.image:
          name: "{{ image_name }}"
          state: present
          container_format: bare
          disk_format: qcow2
          filename: /tmp/{{ qcow2_name }}
          wait: yes
        register: _image_details

      - name: create a bootable volume from image {{ image_name }} in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.volume:
          name: "{{ volume_name }}"
          size: "{{ volume_size }}"
          image: "{{ _image_details.image.id }}"
          state: present
          is_bootable: true
          wait: true
        register: _volume_details

      - name: create the instance {{ instance_name }}  in the openstack project {{ lookup('env', 'OS_PROJECT_NAME') }}
        openstack.cloud.server:
          name: "{{ instance_name }}"
          boot_volume: "{{ _volume_details.volume.id }}"
          flavor: "{{ flavor_name }}"
          nics:
            - port-id: "{{ _port_details.port.id }}"
          key_name: "{{ ssh_key_name }}"
          availability_zone: az1
          timeout: 200
        register: _instance_details
  ```

  # Running playbook to create resources

  The playbook can be run to create the resources

  ```shell
  (genestack) root@bastn:~# source /opt/genestack/scripts/genestack.rc 
  (genestack) root@bastn:~# source user2rc

  (genestack) root@bastn:~# ansible-playbook test1.yml
  [WARNING]: Unable to parse /etc/genestack/inventory as an inventory source
  [WARNING]: No inventory was parsed, only implicit localhost is available
  [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Playbook to create infra on OpenStack flex cloud with ansible]

**********************************************************************************************************************************

TASK [check if the keypair by the name tstk1 already exists]

******************************************************************************************************************************************
ok: [localhost]

TASK [if required create a keypair for the instances]

*************************************************************************************************************************************************
skipping: [localhost]

TASK [Obtain the details of the existing instances]

***************************************************************************************************************************************************
ok: [localhost]

TASK [Fail if there is an existing by the name test12345 in the same project]

*************************************************************************************************************************
skipping: [localhost]

TASK [create a keypair in the openstack project project2]

*********************************************************************************************************************************************
changed: [localhost]

TASK [create the network net8 in the openstack project project2]

**************************************************************************************************************************************
changed: [localhost]

TASK [create the subnet subnet8 for network net8 in the openstack project project2]

*******************************************************************************************************************
changed: [localhost]

TASK [create the security group secgroup1 in the openstack project project2]

**************************************************************************************************************************
changed: [localhost]

TASK [create the router router2 in the openstack project project2]

************************************************************************************************************************************
changed: [localhost]

TASK [create the port port1 in the openstack project project2]

****************************************************************************************************************************************
changed: [localhost]

TASK [if required fetch ubuntu20 cloud image]

*********************************************************************************************************************************************************
changed: [localhost]

TASK [create the image ubuntu20-test1 in the openstack project project2]

******************************************************************************************************************************
changed: [localhost]

TASK [create a bootable volume from image ubuntu20-test1 in the openstack project project2]

***********************************************************************************************************
changed: [localhost]

TASK [create the instance test12345  in the openstack project project2]

*******************************************************************************************************************************
changed: [localhost]


PLAY RECAP
********************************************************************************************************************************************************************************************
localhost                  : ok=13   changed=10   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```
 


  
