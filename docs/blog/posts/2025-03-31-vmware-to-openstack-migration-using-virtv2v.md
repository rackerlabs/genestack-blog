---
date: 2025-03-31
title: VMware to OpenStack Migration using virt-v2v
authors:
  - pram0596
description: >
  Migrate a virtual machine from VMware to OpenStack using vpx.
categories:
  - virt-v2v
  - openstack
---

# VMware to OpenStack Migration using virt-v2v  

This document describes the path to migrate a virtual machine from VMware to OpenStack using virt-v2v vpx. You should use vddk plugins to make this process fast for which link is mentioned in the doc.  

**Pre-requisite:**  
+ Port `5000` should connect from v2v appliance to OpenStack keystone endpoint.  
+ Ports `443,5480` should connect from v2v appliance to VMware vCenter and Esxi hosts.  
+ DNS should resolve the VMware hostnames inside the v2v virtual appliance.  

**Environment:**  
+ **VMware Cloud** - `demo-vmware-cloud.com`  
+ **OpenStack Cloud keystone public endpoint** - `192.168.10.11`  
+ **virt-v2v Virtual appliance** - `192.168.11.11`  

<!-- more -->

**For windows VM migration we need to complete one time additional pre-requites on v2v virtual appliance mentioned on the below link.**  

Coming soon…

**If you want to use vddk plugins then you need to enable those explicitly using link mentioned below.**  

Coming soon…

**Steps:**  
+ **Create v2v appliance for migration on destination OpenStack Cloud.**  
```shell
[root@controller01~]# openstack server create --network NET01 --image ubuntu24 --flavor mig-testing --security-group mig-testing --key-name key01 virt-v2v-appl  
[root@controller01~]# openstack server list  
```

+ **Login to the created virtual appliance and install required package.**  
```shell
[root@controller01~]# ssh -i .ssh/key01ubuntu@192.168.11.11  
[root@virt-v2v-appl]$ sudo -i  
[root@virt-v2v-appl]# apt-get update  
[root@virt-v2v-appl]# apt-get upgrade  
[root@virt-v2v-appl]# reboot  
[root@controller01~]# ssh -i .ssh/key01ubuntu@192.168.11.11  
[root@virt-v2v-appl]# apt-get install virt-v2v -y  
[root@virt-v2v-appl]# apt-get install python3-openstackclient -y  
[root@virt-v2v-appl]# apt install libvirt-clients -y  
```

+ **Check if required ports are opened from virtual appliance.**  
```shell
[root@virt-v2v-appl]# nc -vz 192.168.10.11 5000  
[root@virt-v2v-appl]# nc -vz demo-vmware-cloud.com 443  
[root@virt-v2v-appl]# nc -vz demo-vmware-cloud.com 5480  
```

+ **Copy ca certificate from controller node to virtual appliance.**  
```shell
[root@controller01~]# scp -i .ssh/key01/etc/ssl/certs/ca-certificates.crt ubuntu@192.168.11.11:/tmp/  
```

+ **Move ca certificate under certs directory on the virtual appliance.**  
```shell
[root@controller01~]# ssh -i .ssh/key01ubuntu@192.168.11.11  
root@virt-v2v-appl# mv /tmp/ca-certificates.crt /etc/ssl/certs/  
```

+ **Update bashrc file on virtual appliance to update common OpenStack env variables to connect to destination OpenStack Cloud.**  
```shell
[root@virt-v2v-appl]# vim  /root/.bashrc  
# COMMON OPENSTACK ENVS  
export OS_USERNAME=admin  
export OS_PASSWORD='xxxxxxxxxxxxxxxxxxxxxxxxxxx'  
export OS_PROJECT_NAME=admin  
export OS_TENANT_NAME=admin  
export OS_AUTH_TYPE=password  
export OS_AUTH_URL=https://192.168.10.11:5000/v3  
export OS_NO_CACHE=1  
export OS_USER_DOMAIN_NAME=Default  
export OS_PROJECT_DOMAIN_NAME=Default  
export OS_REGION_NAME=RegionOne  
# For openstackclient  
export OS_IDENTITY_API_VERSION=3  
export OS_AUTH_VERSION=3  
export OS_CACERT=/etc/ssl/certs/ca-certificates.crt  

root@[virt-v2v-appl]# source /root/.bashrc  
root@[virt-v2v-appl]# openstack server list  
```

+ **Run below command to list guests from source VMware cloud. VPX link can be created using below settings**  
`vpx://vcenter.fqdn/datacentername/clustername/hypervisorname?no_verify=1`

no_verify value could be `0` or `1`. If value is `1` it means that SSL verification would be disabled.  

```shell
[root@virt-v2v-appl]# virsh -c 'vpx://demo-vmware-cloud.com/DC1/DC1-Cluster-02/demo-hyp1-cloud.com?no_verify=1' list --all  
Enter username for demo-vmware-cloud.com [administrator@vsphere.local]: domain\user-id  
Enter Domain\user-id's password for demo-vmware-cloud.com:  
Id      Name                                        State
-----------------------------------------------------------------
22      abc1                                        running
43      abc2                                        running
1190    xyz1                                        running
2142    xyz2                                        running
2144    demo1                                       running
7018    demo2                                       running  
```

+ **Run below command to move disk from source and upload to OpenStack volume. Before executing the command make sure that VM is in shutdown state.**  

+ **In the command:**  
`ubuntu20-mig` - Guest VM name on VMware cloud to be migrated  
`password.txt` - Password file created for the VMware domain user on v2v virtual appliance  
`verify-server-certificate=false`  
`server-id` - virt-v2v virtual appliance id running on OpenStack

```shell
[root@virt-v2v-appl:~]# virt-v2v -ic 'vpx://user-id@demo-vmware-cloud.com/DC1/DC1-Cluster-02/demo-hyp1-cloud.com?no_verify=1' ubuntu20-mig -o openstack -ip password.txt -oo verify-server-certificate=false -oo server-id=45dgftbbfddr6784fhskkei8v8483k'  
[   0.0] Setting up the source: -i libvirt -ic vpx://user-id@demo-vmware-cloud.com/DC1/DC1-Cluster-02/demo-hyp1-cloud.com?no_verify=1 ubuntu20-mig
[   6.5] Opening the source
[  89.0] Inspecting the source
[ 561.5] Checking for sufficient free disk space in the guest
[ 561.5] Converting Ubuntu20-mig to run on KVM
virt-v2v: The QEMU Guest Agent will be installed for this guest at first 
boot.
virt-v2v: This guest has virtio drivers installed.
[1885.9] Mapping filesystem data to avoid copying unused and blank areas
[1987.6] Closing the overlay
[1987.8] Assigning disks to buses
[1987.8] Checking if the guest needs BIOS or UEFI to boot
virt-v2v: This guest requires UEFI on the target to boot.
[1987.8] Setting up the destination: -o openstack -oo server-id=kk57djj4yuu589037fkkhe5ii3jk
[2004.1] Copying disk 1/1
█ 100% [****************************************]
[3468.5] Creating output metadata
[3475.6] Finishing off
```

+ **If you want to use vddk plugins then execute the steps mentioned in below link and come back here for further steps to be executed.**  

Coming soon…

+ **You will see a OpenStack volume created on destination OpenStack Cloud.**  
```shell
[root@virt-v2v-appl:~]# openstack volume show rfhyr4565-jj8884j-46d9vj-jjkkrmmchd --fit  
+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Field                          | Value                                                                                                                            |
+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------+
| attachments                    | []                                                                                                                               |
| availability_zone              | nova                                                                                                                             |
| bootable                       | true                                                                                                                             |
| consistencygroup_id            | None                                                                                                                             |
| created_at                     | 2024-11-22T09:31:02.000000                                                                                                       |
| description                    | ubuntu20-mig disk 1/1 converted by virt-v2v                                                                             |
| encrypted                      | False                                                                                                                            |
| id                             | rfhyr4565-jj8884j-46d9vj-jjkkrmmchd                                                                                             |
| migration_status               | None                                                                                                                             |
| multiattach                    | False                                                                                                                            |
| name                           | ubuntu20-mig                                                                                                        |
| os-vol-host-attr:host          | compute02.openstack.cloud.com@ceph#ceph                                                                                  |
| os-vol-mig-status-attr:migstat | None                                                                                                                             |
| os-vol-mig-status-attr:name_id | None                                                                                                                             |
| os-vol-tenant-attr:tenant_id   | rfhyr4565-jj8884j-46d9vj-jjkkrmmchd                                                                                                |
| properties                     | virt_v2v_conversion_date='2024/11/22 08:57:51', virt_v2v_disk_index='1/1', virt_v2v_guest_name='ubuntu20-mig',          |
|                                | virt_v2v_version='2.4.0'                                                                                                         |
| replication_status             | None                                                                                                                             |
| size                           | 12                                                                                                                               |
| snapshot_id                    | None                                                                                                                             |
| source_volid                   | None                                                                                                                             |
| status                         | available                                                                                                                        |
| type                           | __DEFAULT__                                                                                                                      |
| updated_at                     | 2024-11-22T09:55:50.000000                                                                                                       |
| user_id                        | hhfh4i892hjhh5824dkjhaafop49999845jhddg                                                                                                 |
| volume_image_metadata          | {'architecture': 'x86_64', 'hypervisor_type': 'kvm', 'vm_mode': 'hvm', 'hw_disk_bus': 'virtio', 'hw_vif_model': 'virtio',        |
|                                | 'hw_video_model': 'vga', 'hw_machine_type': 'q35', 'os_type': 'linux', 'os_distro': 'ubuntu', 'hw_cpu_sockets': '1',             |
|                                | 'hw_cpu_cores': '2', 'os_version': '20.04', 'hw_rng_model': 'virtio', 'hw_firmware_type': 'uefi'}                                    |
+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------+
```

+ **If source VM has uefi firmware set with secure boot enabled then you need to set boot flag on OpenStack volume additionally.**  
```shell
[root@virt-v2v-appl:~]# openstack volume set --property os_secure_boot=required rfhyr4565-jj8884j-46d9vj-jjkkrmmchd  

[root@virt-v2v-appl:~]# openstack volume show rfhyr4565-jj8884j-46d9vj-jjkkrmmchd --fit
+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Field                          | Value                                                                                                                            |
+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------+
| attachments                    | []                                                                                                                               |
| availability_zone              | nova                                                                                                                             |
| bootable                       | true                                                                                                                             |
| consistencygroup_id            | None                                                                                                                             |
| created_at                     | 2024-11-22T09:31:02.000000                                                                                                       |
| description                    | ubuntu20-mig          disk 1/1 converted by virt-v2v                                                                             |
| encrypted                      | False                                                                                                                            |
| id                             | rfhyr4565-jj8884j-46d9vj-jjkkrmmchd                                                                                             |
| migration_status               | None                                                                                                                             |
| multiattach                    | False                                                                                                                            |
| name                           | ubuntu20-mig-sda                                                                                                                |
| os-vol-host-attr:host          | compute02.openstack.cloud.com@ceph#ceph                                                                                  |
| os-vol-mig-status-attr:migstat | None                                                                                                                             |
| os-vol-mig-status-attr:name_id | None                                                                                                                             |
| os-vol-tenant-attr:tenant_id   | rfhyr4565-jj8884j-46d9vj-jjkkrmmchd                                                                                                |
| properties                     | os_secure_boot='required', virt_v2v_conversion_date='2024/11/22 08:57:51', virt_v2v_disk_index='1/1', virt_v2v_guest_name='ubuntu20-mig' |
|                                | ubuntu20-mig', virt_v2v_version='2.4.0'                                                                                      |
| replication_status             | None                                                                                                                             |
| size                           | 12                                                                                                                               |
| snapshot_id                    | None                                                                                                                             |
| source_volid                   | None                                                                                                                             |
| status                         | available                                                                                                                        |
| type                           | __DEFAULT__                                                                                                                      |
| updated_at                     | 2024-11-22T10:23:39.000000                                                                                                       |
| user_id                        | hhfh4i892hjhh5824dkjhaafop49999845jhddg                                                                                                 |
| volume_image_metadata          | {'architecture': 'x86_64', 'hypervisor_type': 'kvm', 'vm_mode': 'hvm', 'hw_disk_bus': 'virtio', 'hw_vif_model': 'virtio',        |
|                                | 'hw_video_model': 'vga', 'hw_machine_type': 'q35', 'os_type': 'linux', 'os_distro': 'ubuntu', 'hw_cpu_sockets': '1',             |
|                                | 'hw_cpu_cores': '2', 'os_version': '20.04', 'hw_rng_model': 'virtio', 'hw_firmware_type': 'uefi'}                                    |
+--------------------------------+----------------------------------------------------------------------------------------------------------------------------------+
```

+ **Create a new instance using the migrated volume.**  
```shell
[root@virt-v2v-appl:~]# openstack server create --network NET02 --flavor m1.small --security-group mig-testing --key-name key01--volume rfhyr4565-jj8884j-46d9vj-jjkkrmmchd ubuntu20-mig 
+--------------------------------------+-------------------------------------------------+
| Field                                | Value                                           |
+--------------------------------------+-------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                          |
| OS-EXT-AZ:availability_zone          |                                                 |
| OS-EXT-SRV-ATTR:host                 | None                                            |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | None                                            |
| OS-EXT-SRV-ATTR:instance_name        |                                                 |
| OS-EXT-STS:power_state               | NOSTATE                                         |
| OS-EXT-STS:task_state                | scheduling                                      |
| OS-EXT-STS:vm_state                  | building                                        |
| OS-SRV-USG:launched_at               | None                                            |
| OS-SRV-USG:terminated_at             | None                                            |
| accessIPv4                           |                                                 |
| accessIPv6                           |                                                 |
| addresses                            |                                                 |
| adminPass                            | Ggdder5632hy                                    |
| config_drive                         |                                                 |
| created                              | 2024-11-22T10:25:01Z                            |
| flavor                               | m1.small (oowuc9995mmchelkjf098345jkjhhj3) |
| hostId                               |                                                 |
| id                                   | 55357fdr-7543-3224-84g3-kkjf677493kkhd            |
| image                                | N/A (booted from volume)                        |
| key_name                             | key01                                    |
| name                                 | ubuntu20-mig                                    |
| os-extended-volumes:volumes_attached | []                                              |
| progress                             | 0                                               |
| project_id                           | rfhyr4565-jj8884j-46d9vj-jjkkrmmchd               |
| properties                           |                                                 |
| security_groups                      | name=499385-hhfu389-9994-2ff4-dj5lxhj49973jjvnbd'     |
| status                               | BUILD                                           |
| updated                              | 2024-11-22T10:25:01Z                            |
| user_id                              | hhfh4i892hjhh5824dkjhaafop49999845jhddg                |
+--------------------------------------+-------------------------------------------------+
```
```bash
[root@virt-v2v-appl:~]# openstack server list  
+--------------------------------------+-----------------------+--------+---------------------------+---------------------------------+-------------+
| ID                                   | Name                  | Status | Networks                  | Image                           | Flavor      |
+--------------------------------------+-----------------------+--------+---------------------------+---------------------------------+-------------+
| 55357fdr-7543-3224-84g3-kkjf677493kkhd | ubuntu20-mig          | ACTIVE | NET02=10.240.20.24 | N/A (booted from volume)        | m1.small    |

[root@virt-v2v-appl:~]# ping 10.240.20.24  
PING 10.240.20.24 (10.240.20.24) 56(84) bytes of data.
64 bytes from 10.240.20.24: icmp_seq=1 ttl=64 time=3.59 ms
64 bytes from 10.240.20.24: icmp_seq=2 ttl=64 time=1.08 ms
^C
--- 10.240.20.24 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.079/2.334/3.590/1.255 ms
```

+ **Login to the VM with username created on source and move 99-installer.cfg to allow cloud-init to update the config and ssh keys.**  
```shell
[root@virt-v2v-appl:~]# ssh user-id@10.240.20.24
[root@ubuntu20-mig:~]# mv /etc/cloud/cloud.cfg.d/99-installer.cfg /etc/cloud/cloud.cfg.d/99-installer.cfg.bak
[root@ubuntu20-mig:~]# reboot
```

+ **Now you will be able to login using default cloud username and ssh keys.**  
```shell
[root@virt-v2v-appl:~]# ssh -i .ssh/key01 ubuntu@10.240.20.24
```

+ **Remove VMware Tools from the migrated instance.**  
```shell
[root@ubuntu20-mig:~]# apt-get remove --purge open-vm-tools
```
