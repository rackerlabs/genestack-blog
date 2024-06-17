---
date: 2024-06-13
title: Creating a WordPress site using Openstack-Flex
authors:
  - dereknoblej
description: >
  Creating a WordPress site using Openstack-Flex
categories:
  - General
  - Wordpress
  - Application
  - Server
---

# Creating a WordPress site using Openstack-Flex

Here I wanted to show our customers and hopefully future customers on how easy it is to host your own Wordpress site using Openstack Flex. 

<!-- more -->

## Getting Started:

First you will need to set up your clouds.yaml file to be able to complete the next steps. More information about that can be found [here](https://docs.rackspacecloud.com/build-test-envs/#configure-openstack-client). 

### Deploying Our Openstack Server

In this section I will provide examples on how I was able to set up my Openstack Server these are just examples please feel free to set up your environment as you would like.

First we need to create our router. I am naming mine wordpress-router.

``` shell
openstack --os-cloud={cloud_name} router create wordpress-router
```

Second we create our network which I will name wordpress-network.

``` shell
openstack --os-cloud {cloud_name} network create wordpress-network
```

Third we need to set the external gateway on our wordpress-router.

``` shell
openstack --os-cloud {cloud_name} router set --external-gateway PUBLICNET wordpress-router
```

Fourth we are going to set up our subnet, you can choose between ipv4 and ipv6. The ip range is also up to you. For the DNS name server you will need to ping cachens1.sjc3.rackspace.com and cachens2.sjc3.rackspace.com. I am naming my subnet wordpress-subnet.

``` shell
ping cachens1.sjc3.rackspace.com -c2
ping cachens2.sjc3.rackspace.com -c2

openstack --os-cloud {cloud_name} subnet create --ip-version 4 --subnet-range 172.20.108.0/24 --dns-nameserver {ip address} --dns-nameserver {ip address} --network wordpress-network wordpress-subnet
```

Fifth we will add our subnet to our wordpress-router 

``` shell
openstack --os-cloud {cloud_name} router add subnet wordpress-router wordpress-subnet
```

Sixth we are going to add our security group and add our ports of ingress we will need to allow for HTTP, HTTPS and SSH traffic. I am naming my security group wordpress-sg. 

``` shell
openstack --os-cloud {cloud_name} security group rule create --ingress --remote-ip 0.0.0.0/0 --dst-port 80:80 --protocol tcp wordpress-sg
openstack --os-cloud {cloud_name} security group rule create --ingress --remote-ip 0.0.0.0/0 --dst-port 443:443 --protocol tcp wordpress-sg
openstack --os-cloud {cloud_name} security group rule create --ingress --remote-ip 0.0.0.0/0 --dst-port 22 --protocol tcp wordpress-sg
```

Seventh we need to create our floating ip.

``` shell
openstack --os-cloud {cloud_name} floating ip create --subnet PUBLICNET_SUBNET PUBLICNET
```

Eighth we are going to create our Public and Private ssh keys so we can securely connect to our server. I am naming my key wordpress-key

``` shell
ssh-keygen
```
This will prompt you store and name your private key. I did something like this /home/{username}/.ssh/wordpress-key. 

After that we will create our public key using the command below then we will assign it using the the openstack cli tools.

``` shell
ssh-keygen -f ~/.ssh/wordpress-key -y > ~/,ssh/wordpress-key.pub 
openstack —os-cloud {cloud_name} key-pair create --public-key ~/.ssh/wordpress-key.pub wordpress-key
```
Ninth we are ready to create our server! You will be able to customize your flavor, image, volume size. We want to make sure to attach our network, key-name and security group. 

``` shell
openstack --os-cloud {cloud_name} server create --flavor m1.medium --image Ubuntu-22.04 --boot-from-volume 40 --network wordpress-network --key-name wordpress-key --security-group wordpress-sg wordpress-server
```

Lastly we will want to make sure to assign our floating ip. We can do this by adding it to our port for the server. If you get the fixed ip from our newly create server you can find the port ID by searching through the port list. Look for the fixed ip address of our server and grab the corresponding ID.

``` shell
openstack --os-cloud {cloud_name} port list
openstack --os-cloud {cloud_name} floating ip set --port {port id}
```

SSH into your new server! 

``` shell
ssh -i /home/.ssh/wordpress-key ubuntu@{floating ip}
```

### Deploying Our Wordpress site on our Openstack-Flex Server

Once we have created our Openstack-Flex Server we will want to make sure everything is up to date. 

``` shell
sudo apt-get update && sudo apt-get upgrade -y
```

The next step is to install apache and mysql.

``` shell
sudo apt install apache2
```

``` shell
sudo apt install mysql-server
sudo mysql_secure_installation
```

Then we will want to install php and restart apache. 

``` shell
sudo apt install php libapache2-mod-php php-mysql
```

``` shell
sudo systemctl restart apache2
```

Next we need to use mysql and create our wordpress database with our username and password. Please create your own personal username and password. 

``` shell
sudo mysql
```

``` shell
CREATE DATABASE wordpress;
CREATE USER 'openstack-flex'@'localhost' IDENTIFIED BY 'pass';
GRANT ALL PRIVILEGES ON wordpress.* TO 'openstack-flex'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Download Wordpress. 

``` shell
cd /tmp
curl -LO https://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
```

Update our config file, you can find the example in /tmp/wordpress/wp-config-sample.php. Take this example and copy it to make your own config. 

``` shell
cp /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php
```

Using any text editor you are comfortable with update the 'DB_NAME', 'DB_USER', and 'DB_PASSWORD' to the values you created in the database earlier.

Next, we will copy our wordpress files to the apache root directory and set the permissions for the them.

``` shell
sudo rsync -avP /tmp/wordpress/ /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

Don't forget to remove the apache index.html which can be found in /var/www/html.

At this point you should be able to take our floating ip and plug it into your web browser. Then we should be greeted by the WordPress Setup page. 
