---
date: 2024-11-13
title: Using ansible to manage instances on flex cloud
authors:
  - puni4220
description: >
  Using ansible to manage instances on flex cloud
categories:
  - ansible
  - openstack
---
# Using ansible to manage instances on flex cloud

In this blog post we will discuss how we can use ansible to manage instances running on a flex cloud. It is important to note that
while it is possible to create resources on an openstack cloud using ansible itself but the main aim of this blog post is to discuss
how we can manage existing instances running within a project with ansible. The examples provided in this blog post are for instances 
running ubuntu 20.04 LTS as the base OS; these instructions can be adapted to accomodate any other OS as well

The blog post assumes that the the node from where we are running the openstack and ansible commands has network access to the flex cloud
and the credentions for a valid user in the flex cloud are sourced and ansible is installed on the node from where the ansible playbooks and adhoc
commands will run. The [bootstrap script](https://docs.rackspacecloud.com/genestack-getting-started/) in the install guide creates the venv with the
necessary components installed inside the venv

The first step in this process is to gather the required details for generating the ansible inventory for our instances; For the 
example presented in this blog post there are 2 instances which have floating ip(s) assigned to them and the floating ip(s) are reachable
from the bastion host from where the ansible adhoc commands and ansible playbooks will run

+ To generate the inventory for the existing instances within a project capture the list of running instances:
```shell
(genestack) root@bastn:~# openstack server list

+--------------------------------------+--------------+--------+---------------------------------+----------+-----------+

| ID                                   | Name         | Status | Networks                        | Image    | Flavor    |

+--------------------------------------+--------------+--------+---------------------------------+----------+-----------+

| 94e7b2eb-a3b1-4f48-9e86-70f463e7e8bd | test-epsilon | ACTIVE | net1=10.10.10.151, 172.16.8.94  | ubuntu20 | m1.medium |

| 9f8b7596-c332-4000-8154-650d5a2e95fd | test-alpha   | ACTIVE | net1=10.10.10.139, 172.16.8.170 | ubuntu20 | m1.medium |

+--------------------------------------+--------------+--------+---------------------------------+----------+-----------+
```

+ Then we will need the fixed ip(s) assigned to the running instances:
```shell
(genestack) root@bastn:~# for i in $(openstack server list -c ID -f value); do openstack port list --server $i -c "Fixed IP Addresses" -f value; done
[{'subnet_id': 'd31a8c8b-e8f5-47fa-9d2e-0d30df2fd37c', 'ip_address': '172.16.8.94'}]
[{'subnet_id': 'd31a8c8b-e8f5-47fa-9d2e-0d30df2fd37c', 'ip_address': '172.16.8.170'}]
```

+ With the fixed ip(s) for the ports associated with the instances we can capture the floating ip(s) associated with these instances:
```shell
(genestack) root@bastn:~# for i in 172.16.8.94 172.16.8.170; do openstack floating ip list --fixed-ip-address $i -f yaml; done
- Fixed IP Address: 172.16.8.94
  Floating IP Address: 10.10.10.151
  Floating Network: c03d6556-d7cb-44c6-a0cd-2985c47ddcfd
  ID: 2f1163f2-46c7-4092-8755-db0542ec2383
  Port: f6051f7f-c2b4-4803-9453-3ac67da27c55
  Project: 7d9c7a46b38d40b891dd0c40644773ee

- Fixed IP Address: 172.16.8.170
  Floating IP Address: 10.10.10.139
  Floating Network: c03d6556-d7cb-44c6-a0cd-2985c47ddcfd
  ID: 58ff2b9e-1ebb-475a-b60a-f645aa72b75f
  Port: 7fe428be-4fe0-4e09-ae0d-3aea09c1ccf4
  Project: 7d9c7a46b38d40b891dd0c40644773ee
```

+ With the floating ip(s) listed we can create a simple **inventory.ini** inventory file for ansible:
```ini 
[all]

test-alpha ansible_ssh_host=10.10.10.139 ansible_ssh_user=ubuntu

test-epsilon ansible_ssh_host=10.10.10.151 ansible_ssh_user=ubuntu


[alpha]

test-alpha


[epsilon]

test-epsilon
```
It should be noted that in this case the **ansible_ssh_user** is set to **ubuntu** this is because it is the default in the cloud images for Ubuntu OS and the **ubuntu** user has
sudo privileges which are required by ansible to execute the tasks on the instances

+ With the inventory created we can try to run the **ping** module to test whether ansible can reach the instances on the flex cloud:
```shell
(genestack) root@bastn:~# ansible -i inventory.ini epsilon -m ping --private-key ansible-key

test-epsilon | SUCCESS => {

    "ansible_facts": {

        "discovered_interpreter_python": "/usr/bin/python3"

    },

    "changed": false,

    "ping": "pong"

}
```

+ We can see that the ping succeeded and now to install **apache2** package on test-epsilon instance with ansible using an adhoc command:
```shell
(genestack) root@bastn:~# ansible -i inventory epsilon -m apt -a "name=apache2" --private-key ansible-key --become
test-epsilon | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "cache_update_time": 1731470200,
    "cache_updated": false,
    "changed": false
}
```
In this example we can see that we can easily install **apache2** package on the instance named **test-epsilon** by referring to it's group name in the inventory

+ Let's say that we would like to run a full playbook against the instance named **test-alpha** running on the flex cloud; the example playbook in this case installs **apache2** package
on the instance and configures a custom homepage for the guest; for this purpose we will first need to create the playbook and the jinja file:
```yaml title="main.yml"
(genestack) root@bastn:~# cat main.yml
---
- name: Playbook to install apache on Ubuntu instances on flex cloud
  hosts: alpha
  gather_facts: true
  tasks:
    - name: update apt cache on the ubuntu guest
      ansible.builtin.apt:
        update_cache: yes

    - name: install the apache package on the guest
      ansible.builtin.apt:
        name: apache2
        state: present

    - name: enable and start apache2 service on the guest
      ansible.builtin.service:
        name: apache2
        enabled: yes
        state: started

    - name: copy the custom index.html file to the document root on the guest
      template:
        src: index.html.j2
        dest: /var/www/html/index.html
      notify:
        - restart apache2 service

  handlers:
    - name: restart apache2 service
      ansible.builtin.service:
        name: apache2
        state: restarted  
```
```jinja title="index.html.j2"
(genestack) root@bastn:~# cat index.html.j2
<html>

<head>

  <title> ansible on flex cloud </title>

</head>

<body>

  <p> this website is running on {{ ansible_hostname }} ubuntu guest </p>

</body>

</html>
```

+ We can then run the playbook which installs **apache2** package on the test-alpha instances and configures a custom homepage for the webserver running inside the guest:
```shell
(genestack) root@bastn:~# ansible-playbook --inventory inventory.ini main.yml --private-key ansible-key --become

 

PLAY [Playbook to install apache on Ubuntu instances on flex cloud] *******************************************************************************************************************************************

 

TASK [Gathering Facts] ****************************************************************************************************************************************************************************************

ok: [test-alpha]

 

TASK [update apt cache on the ubuntu guest] *******************************************************************************************************************************************************************

changed: [test-alpha]

 

TASK [install the apache package on the guest] ****************************************************************************************************************************************************************

changed: [test-alpha]

 

TASK [enable and start apache2 service on the guest] **********************************************************************************************************************************************************

ok: [test-alpha]

 

TASK [copy the custom index.html file to the document root on the guest] **************************************************************************************************************************************

changed: [test-alpha]

 

RUNNING HANDLER [restart apache2 service] *********************************************************************************************************************************************************************

changed: [test-alpha]

 

PLAY RECAP ****************************************************************************************************************************************************************************************************

test-alpha                 : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
It should be noted that in this case the **ansible-key** is the key with which the instances were created and the same key is provided to ansible
