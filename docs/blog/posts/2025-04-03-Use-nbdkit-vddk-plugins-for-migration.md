---
date: 2025-04-03
title: Use nbdkit vddk plugins for migration
authors:
  - pram0596
description: >
  Use nbdkit vddk plugins for migration from VMware to OpenStack. 
categories:
  - virt-v2v
  - openstack
  - migration
---
# Use nbdkit vddk plugins for migration  
This document describes the method to use nbdkit `vddk` plugins for migration from VMware to OpenStack.  `vddk` plugins extensively makes it quite fast and takes less time to perform data migration. 

## Pre-requisite  
+ Port `902` and `443` should connect from v2v appliance to VMware vCenter and Esxi hosts.  
+ DNS should resolve the VMware `FQDN` inside the v2v virtual appliance.  
+ `nbdkit vddk` plugins should be enabled on the virtual appliance.  
+ `vddk` plugins must have been installed and configured on the virtual appliance. Refer [Enable nbdkit vddk plugins](https://blog.rackspacecloud.com/blog/2025/04/01/enable_nbdkit_vddk_plugins/) to configure it.
## Environment  
Kindly refer below details which is used in this documentation. These IPs, FQDN can be different in your environment.  

+ **VMware Cloud** - `demo-vmware-cloud.com`  
+ **OpenStack Cloud keystone public endpoint** - `192.168.10.11`    
+ **virt-v2v Virtual appliance** - `192.168.11.11`    
## Steps  
### Obtain SSL thumbprint
You are required to obtain SSL thumbprint for the VMware cloud URL. Below is the syntax to get the thumbprint.  
```shell
openssl s_client -connect vcenter.example.com:443</dev/null | openssl x509 -in /dev/stdin -fingerprint -sha1 -noout  
```  
Execute below command from cli to obtain `SHA` thumbprint. 

**On v2v appliance**
```shell
openssl s_client -connect demo-vmware-cloud.com:443</dev/null | openssl x509 -in /dev/stdin -fingerprint -sha1 -noout
```
You will see below output once you execute above command.  

!!! example "Example output"

    ```shell
    depth=0 CN = demo-vmware-cloud.com, C = US, ST = California, L = Palo Alto, O = VMware, OU = VMware Engineering
    verify error:num=20:unable to get local issuer certificate
    verify return:1
    depth=0 CN = demo-vmware-cloud.com, C = US, ST = California, L = Palo Alto, O = VMware, OU = VMware Engineering
    verify error:num=21:unable to verify the first certificate
    verify return:1
    depth=0 CN = demo-vmware-cloud.com, C = US, ST = California, L = Palo Alto, O = VMware, OU = VMware Engineering
    verify return:1
    DONE
    sha1 Fingerprint=34:64:92:85:F5:6G:k3:C4:7E:8Y:2L:E2:F5:65:43:76:2A:A6:4H:3G
    ```  
### Migrate virtual machine  
Below is the command syntax to migrate the virtual machine using vddk plugins.  
```shell
virt-v2v -ic 'vpx://user@vCenter.example.com/datacenter/clustername/hypervisor_fqdn?no_verify=1' \  
            -it vddk -io vddk-libdir=path_to_vmware_vddk_library \  
            -io vddk-thumbprint=xx:xx:xx:xx:xx:xx.... guestname \  
            -o openstack -ip password_file_for_user -oo verify-server-certificate=false \  
            -oo server-id=virtual_appliance_UUID  
```
Excute below command to perfrom the migaration.  

**On v2v appliance**  
```shell
virt-v2v -ic 'vpx://user-id@demo-vmware-cloud.com/DC1/DC1-Cluster-02/demo-hyp1-cloud.com?no_verify=1' \
            -it vddk -io vddk-libdir=/opt/vmware-vix-disklib-distrib/lib64 \
            -io vddk-thumbprint=34:64:92:85:F5:6G:k3:C4:7E:8Y:2L:E2:F5:65:43:76:2A:A6:4H:3G ubuntu20-mig\
            -o openstack -ip password.txt -oo verify-server-certificate=false -oo server-id='45dgftbbfddr6784fhskkei8v8483k'
```
You will notice the below output upon successful execution.  

!!! example "Example output"

    ```shell
    [   0.0] Setting up the source: -i libvirt -ic vpx://user-id@demo-vmware-cloud.com/DC1/DC1-Cluster-02/demo-hyp1-cloud.com?no_verify=1 -it vddk ubuntu20-mig
    [   4.5] Opening the source
    [  28.0] Inspecting the source
    [  46.7] Checking for sufficient free disk space in the guest
    [  46.7] Converting ubuntu release 22 to run on KVM
    virt-v2v: The QEMU Guest Agent will be installed for this guest at first 
    boot.
    virt-v2v: This guest has virtio drivers installed.
    [ 210.5] Mapping filesystem data to avoid copying unused and blank areas
    [ 212.1] Closing the overlay
    [ 212.3] Assigning disks to buses
    [ 212.3] Checking if the guest needs BIOS or UEFI to boot
    virt-v2v: This guest requires UEFI on the target to boot.
    [ 212.3] Setting up the destination: -o openstack -oo server-id=45dgftbbfddr6784fhskkei8v8483k
    [ 228.9] Copying disk 1/1
    █ 100% [****************************************]
    [ 512.0] Creating output metadata
    [ 517.4] Finishing off
    ```  

Follow further steps mentioned in doc [VMware to OpenStack Migration using virt-v2v](https://blog.rackspacecloud.com/blog/2025/04/01/vmware_to_openstack_migration_using_virt-v2v/) to create instance on migrated volume.
