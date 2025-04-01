---
date: 2025-04-01
title: Enable nbdkit vddk plugins
authors:
  - pram0596
description: >
  Enable vddk plugins to migrate a virtual machine from VMware to OpenStack using vpx.
categories:
  - virt-v2v
  - openstack
  - migration
---

# Enable nbdkit vddk plugins 
 
This document describes the path to build and install vddk plugins for nbdkit which is required to migrate a virtual machine from VMware to OpenStack using vpx. Please keep in mind that it requires VMware proprietary library that you must download yourself.  

## Pre-requisite:  
+ One should have account to Broadcom developer site to download library.  

## Environment:  
+ **virt-v2v Virtual appliance** - `192.168.11.11`  

## Reference Docs:  
https://libguestfs.org/nbdkit-vddk-plugin.1.html  
https://developer.broadcom.com/sdks/vmware-virtual-disk-development-kit-vddk/latest  
https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-vddk-programming-guide/GUID-158D8330-7C9C-4F9C-83E1-4DC154DA3C66.html  

## Steps:   
We can divide the whole procedure into two parts. First would be to build and install vddk plugins for nbdkit and second would be to download and use VMware vddk library.

### Build and Install vddk plugins:  
+ #### Login to virt-v2v appliance.  
#### On controller node:    
```shell
ssh -i ~/.ssh/rpc_support ubuntu@192.168.11.11  
sudo -i  
```  

+ #### Download vddk code from git repository.
#### On virtual appliance:
```shell 
git clone https://github.com/libguestfs/nbdkit.git
```  
+ #### Run below commands to install required packages and vddk plugins.
#### On virtual appliance:
```shell
sudo apt-get install libtool
apt install pkg-config m4 libtool automake autoconf
cd nbdkit/
```
```shell
autoreconf -i
./configure --disable-dependency-tracking
```
```shell
apt install make 
make
make install
```
```shell
cp /usr/local/lib/nbdkit/plugins/nbdkit-vddk-plugin.so \
   /usr/lib/x86_64-linux-gnu/nbdkit/plugins/  
```
+ #### Check if vddk plugins has been installed successfully.  
#### On virtual appliance:  
```shell
nbdkit vddk libdir=/usr/local/lib/nbdkit/plugins/ --dump-plugin  
```
#### Example output:
```shell
path=/usr/local/lib/nbdkit/plugins/nbdkit-vddk-plugin.so
name=vddk
version=1.33.11
api_version=2
struct_size=384
max_thread_model=parallel
thread_model=parallel
errno_is_preserved=0
magic_config_key=file
has_longname=1
has_unload=1
has_dump_plugin=1
has_config=1
has_config_complete=1
has_config_help=1
has_get_ready=1
has_after_fork=1
has_open=1
has_close=1
has_get_size=1
has_block_size=1
has_can_flush=1
has_can_extents=1
has_can_fua=1
has_pread=1
has_pwrite=1
has_flush=1
has_extents=1
vddk_default_libdir=/usr/local/lib/vmware-vix-disklib
vddk_has_nfchostport=1
```  
+ #### Download and place VMware vddk library:  
Login to Broadcom web link https://developer.broadcom.com/sdks/vmware-virtual-disk-development-kit-vddk/latest and download zip file. Place it under /tmp/ on virt-v2v appliance. In my case I downloaded the latest version which is `8.0.3`.

+ #### Move tar file under /opt and extract it.
```shell
ls -l /tmp/VMware-vix-disklib-8.0.0-20521017.x86_64.tar.gz
mv /tmp/VMware-vix-disklib-8.0.0-20521017.x86_64.tar.gz /opt/
```
```shell
cd /opt
tar xvzf VMware-vix-disklib-8.0.0-20521017.x86_64.tar.gz 
cd vmware-vix-disklib-distrib/
ls
```
+ #### Appliance is ready to migrate the vm using vddk plugins now.
