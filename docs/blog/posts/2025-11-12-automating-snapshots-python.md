---
date: 2025-11-12
title: Automating Scheduled VM Snapshots
authors:
  - brianabshier
description: >
  Automating VM Snapshotting with OpenStack Flex VMs
categories:
  - snapshots
  - applicationcredentials
  - openstack
---

This guide walks through creating an OpenStack **Application Credential**, setting up a Python environment, installing the snapshot script, and scheduling it to run automatically.

<!-- more -->

# OpenStack VM Snapshot Automation Setup Guide

## 1. Create an Application Credential

1. Log in to the Skyline panel for **Rackspace OpenStack Flex** as shown [here](https://docs.rackspace.com/docs/accessing-rackspace-flex-cloud)
2. Create your Application Credential with 'creator' role as seen [here](https://docs.rackspace.com/docs/application-credentials-in-openstack-flex)
3. After creation, a `.json` file will be automatically downloaded — **save this file securely**.

   * This file contains your **Application Credential ID** and **Secret**, which are required for authentication.

---

## 2. Create the Virtual Machine (Example: Ubuntu 24.04)

In this example we'll have our snapshot automation script running on a VM so that we don't need to worry about leaving a workstation or something running.

Launch a new VM running **Ubuntu 24.04 LTS** through the OpenStack Dashboard or CLI. You can see our guide on this [here](https://docs.rackspace.com/docs/creating-a-virtual-machine)

Once your VM is active, connect via SSH.

>**NOTE:** Your VM does not need a Floating IP to perform the snapshot functions, but if it doesn't have a Floating IP you'll need to SSH into it from another VM on the same private network.

---

## 3. Prepare the Python Environment

Run the following commands to install dependencies and create a virtual environment:

```bash
python3 --version
sudo apt install python3-pip -y
sudo apt install python3-openstacksdk -y
sudo apt install python3.12-venv -y

sudo python3 -m venv ~/venv/openstack
source ~/venv/openstack/bin/activate

pip install --upgrade pip
pip install openstacksdk
```

---

## 4. Install the Snapshot Script

Download the snapshot management script:

```bash
wget https://raw.githubusercontent.com/brianabshier/python_scripts/main/snapshot_vms.py -O ~/snapshot_vms.py
```

Update the script to include your **Application Credential** values from the downloaded JSON file. In my example I'm using the DFW region for the endpoint - but change it to the region you need:

```python
# --- CONFIGURATION ---
AUTH_URL = "$https://keystone.api.dfw3.rackspacecloud.com/v3"
APP_CRED_ID = "$APPLICATION_CRED_ID"
APP_CRED_SECRET = "$APPLICATION_CRED_SECRET"
```

Make the script executable:

```bash
chmod +x ~/snapshot_vms.py
```

---

## 5. Excluding VMs from Snapshot

Inside the `.snapshot_data` directory, you can create the following files to exclude specific instances:

* `exclude_ids.txt` — one instance ID per line
* `exclude_names.txt` — one instance name per line

Example:

```bash
mkdir -p ~/.snapshot_data
echo "vm-name-to-skip" >> ~/.snapshot_data/exclude_names.txt
```

---

## 6. Schedule Automatic Snapshots (Cron)

To run the script daily at **2:30 AM UTC**, edit your crontab:

```bash
crontab -e
```

Add the following line:

```bash
30 2 * * * /usr/bin/python3 /home/ubuntu/snapshot_vms.py >> /home/ubuntu/.snapshot_data/snapshot.log 2>&1
```

> **Note:** If your server uses UTC, the job will run at 2:30 AM UTC. Adjust the schedule if needed.

---

## 7. Log and Output

All logs will be written to:

```
~/.snapshot_data/snapshot.log
```

You can monitor output with:

```bash
tail -f ~/.snapshot_data/snapshot.log
```

---

### Summary

* Created OpenStack **Application Credential (creator role)**
* Set up Python **virtual environment**
* Installed **snapshot_vms.py**
* Configured **exclusions** for VMs
* Scheduled **automated daily execution**
