---
date: 2025-04-07
title: virt-v2v Windows VM migration pre-requisite
authors:
  - pram0596
description: >
  This document describes the pre-requisites needed to complete a windows VM machine migration from VMware cloud to OpenStack.
categories:
  - virt-v2v
  - openstack
  - migration
---  
# `virt-v2v` `Windows` VM migration pre-requisite  

This article explains the prerequisites for migrating a `Windows2019` VM from `VMware cloud` to `OpenStack`. These are additional requirements that needs to be setup before completing a `Windows` VM migration. If you do not complete it, you may see following error while completing a `Windows` VM migration.  

```shell
virt-v2v: error: One of rhsrvany.exe or pvvxsvc.exe is missing in /usr/share/virt-tools.
One of them is required in order to install Windows firstboot scripts.
You can get one by building rhsrvany (https://github.com/rwmjones/rhsrvany)
```  
## Pre-requisite  

+ `virt-v2v` appliance must be configured as per [VMware to OpenStack Migration using virt-v2v](https://blog.rackspacecloud.com/blog/2025/04/01/vmware_to_openstack_migration_using_virt-v2v/)  

## Environment  

Kindly refer below details which is used in this documentation. IP can be different in your environment.  

+ **virt-v2v Virtual appliance** - `192.168.11.11`  

## Steps  
Perform all below listed steps on virtual appliance to confiure it to be able to migrate `windows` VM.  

### Download virtio iso image  

Download `virtio` iso image on virtual appliance. `virtio` image version can differ for different `windows` OS flavor so please check `virtio` compatibility matrix before downloading. Here I am using version `0.1.248` which is compatible with `windows19`.   
```shell
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.248-1/virtio-win-0.1.248.iso
```  

### Mount downloaded image to `/mnt` directory  

Mount iso image to `/mnt` directory.  
```shell
mount virtio-win-0.1.248.iso /mnt
```
### Create `virtio-win` directory  

Create `virtio-win` directory under `/usr/share`  
```shell
mkdir /usr/share/virtio-win
```
### Copy image data  

Switch to `/mnt` directory and copy data to `/usr/share/virtio-win`.  
```shell
cd /mnt
cp -rpv * /usr/share/virtio-win/
cd
umount /mnt
ls -l /usr/share/virtio-win/ 
```  
Check if you can see below files under `/usr/share/virtio-win/`

!!! example "Example output"

    ```shell
    total 749040
    dr-xr-xr-x 16 root root      4096 Feb 28  2024 Balloon
    dr-xr-xr-x 16 root root      4096 Feb 28  2024 NetKVM
    dr-xr-xr-x 13 root root      4096 Feb 28  2024 amd64
    dr-xr-xr-x  2 root root      4096 Feb 28  2024 cert
    dr-xr-xr-x  2 root root      4096 Feb 28  2024 data
    dr-xr-xr-x 11 root root      4096 Feb 28  2024 fwcfg
    dr-xr-xr-x  2 root root      4096 Feb 28  2024 guest-agent
    dr-xr-xr-x  6 root root      4096 Feb 28  2024 i386
    dr-xr-xr-x 14 root root      4096 Feb 28  2024 pvpanic
    dr-xr-xr-x  7 root root      4096 Feb 28  2024 qemufwcfg
    dr-xr-xr-x 14 root root      4096 Feb 28  2024 qemupciserial
    dr-xr-xr-x  5 root root      4096 Feb 28  2024 qxl
    dr-xr-xr-x  9 root root      4096 Feb 28  2024 qxldod
    dr-xr-xr-x  5 root root      4096 Feb 28  2024 smbus
    dr-xr-xr-x 11 root root      4096 Feb 28  2024 sriov
    dr-xr-xr-x 11 root root      4096 Feb 28  2024 viofs
    dr-xr-xr-x 11 root root      4096 Feb 28  2024 viogpudo
    dr-xr-xr-x 13 root root      4096 Feb 28  2024 vioinput
    dr-xr-xr-x 14 root root      4096 Feb 28  2024 viorng
    dr-xr-xr-x 14 root root      4096 Feb 28  2024 vioscsi
    dr-xr-xr-x 16 root root      4096 Feb 28  2024 vioserial
    dr-xr-xr-x 16 root root      4096 Feb 28  2024 viostor
    -r-xr-xr-x  1 root root   4539904 Feb 28  2024 virtio-win-gt-x64.msi
    -r-xr-xr-x  1 root root   2573312 Feb 28  2024 virtio-win-gt-x86.msi
    -r-xr-xr-x  1 root root  27447293 Feb 28  2024 virtio-win-guest-tools.exe
    -r-xr-xr-x  1 root root      1598 Feb 28  2024 virtio-win_license.txt
    ```  
### Install additional packages  

Run below commands to install few more packages needed for `windows` migration.
```shell
apt install -y rpm2cpio
wget -nd -O srvany.rpm https://kojipkgs.fedoraproject.org//packages/mingw-srvany/1.1/4.fc38/noarch/mingw32-srvany-1.1-4.fc38.noarch.rpm
rpm2cpio srvany.rpm | cpio -idmv \
  && mkdir /usr/share/virt-tools \
  && mv ./usr/i686-w64-mingw32/sys-root/mingw/bin/*exe /usr/share/virt-tools/
```
Appliance is ready to perform `windows` vm migration. Switch back to doc [VMware to OpenStack Migration using virt-v2v](https://blog.rackspacecloud.com/blog/2025/04/01/vmware_to_openstack_migration_using_virt-v2v/) for further steps to perform actual disk migration.  
