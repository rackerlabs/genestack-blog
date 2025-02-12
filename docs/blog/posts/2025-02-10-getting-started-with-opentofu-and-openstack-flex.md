---
title: Getting Started with OpenTofu and OpenStack Flex
date: 2025-02-10
authors:
  - cblument
description: Getting Started with OpenTofu and OpenStack Flex
categories:
  - Automation
  - DevOps
  - OpenTofu
  - Terraform
---

# Getting Started with OpenTofu and OpenStack Flex

OpenTofu is an infrastructure as code tool that lets you define both cloud and on-prem resources in human-readable configuration files that you can version, reuse, and share.

We will demonstrate using OpenTofu to build a three node environment. The environment will consist bastion server to connect in via ssh, a webserver to serve web content and a database server. An OpenStack account with appropriate permissions will be needed to build the environment.

<!-- more -->

!!! note
    This post should also apply to Hashicorp Terraform as well. At the end when applying the definitions the command `terraform` instead of `tofu` can be used.

## Installing OpenTofu

Installation details can be found at the [official website](https://opentofu.org/docs/intro/install/).

### MacOS and homebrew

To quickly get OpenTofu installed on a mac:

```bash
brew upate
brew install opentofu
```

## Configuring OpenTofu for Openstack using clouds.yaml

The `~/.config/openstack/clouds.yaml` file needs to be populated with your credentials to access openstack-flex. This allows for easily using the openstack cli as well as OpenTofu for configuration.

```yaml
clouds:
  openstack-flex:
    auth_url: "AUTH_URL"
    project_name: "PROJECT_NAME"
    username: "USERNAME"
    password: "PASSWORD"
    region_name: "REGION"

```

## Declaring your resources in openstack-flex

The OpenTofu language is used to declare resources and in this case those resources are in openstack-flex. The rest of the code snippets must go into the `main.tf` file.

## Define variables

These variables will be used later. Start by creating a file named `main.tf`. Later these variables will be used in resources. Notice that two variables are being created:

- cloud
- ssh-public-key-path

These variables have default values assigned and can be overriden in the cli.

```
variable "cloud" {
  type        = string
  default = "openstack-flex"
  description = "The cloud configuration to use from clouds.yaml"
  sensitive   = false
}

variable "ssh-public-key-path" {
  type = string
  default = "~/.ssh/id_rsa.pub"
  description = "path to public key you would like to add to openstack"
  sensitive   = false
}
```

## Configure openstack provider

OpenTofu needs to include the [openstack provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs) from the provider registry.

Rather than specify the openstack endpoint details in the provider config the `clouds.yaml` config setup earlier can be used. This defaults to the variable `cloud` setup above. It can be overriden specifying a different string on the cli such as `tofu plan -var "cloud=[ CLOUD NAME ]`.

```
terraform {
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 3.0.0"
    }
  }
}

provider "openstack" {
  cloud = var.cloud
}
```

## Configure network in openstack.

The network resources need to be created for the compute instances. Below is a brief description of each resource followed by the OpenTofu code that builds the networking resources.

- Get the external network

The external network in openstack-flex is `PUBLiCNET`. This network is needed to create the `external-network` resource for connectivity. In this case using the data block retrieves external network to be used by the `external-router` resource.

- Create a router `external-router` and attaches to the external network
- Create an network `internal-network`
- Create a subnet `internal-subnet` associating it with the network `internal-network`.
- Create the internal router interface `internal-router-interface` that links the `internal-subnet` to the `external-router`

```
## Get the external network
data "openstack_networking_network_v2" "external-network" {
  name = "PUBLICNET"
}

## Create a router
resource "openstack_networking_router_v2" "external-router" {
  name                = "external-router"
  admin_state_up      = true
  external_network_id = data.openstack_networking_network_v2.external-network.id
}

## Create internal network
resource "openstack_networking_network_v2" "internal-network" {
  name                  = "internal-network"
  admin_state_up        = "true"
  external              = false
  port_security_enabled = true
}

## Create internal subnet
resource "openstack_networking_subnet_v2" "internal-subnet" {
  name        = "internal-subnet"
  network_id  = openstack_networking_network_v2.internal-network.id
  cidr        = "192.168.50.0/24"
  ip_version  = 4
  enable_dhcp = true
  allocation_pool {
    start = "192.168.50.10"
    end   = "192.168.50.254"
  }
}

## Create internal router interface
resource "openstack_networking_router_interface_v2" "internal-router-interface" {
  router_id = openstack_networking_router_v2.external-router.id
  subnet_id = openstack_networking_subnet_v2.internal-subnet.id
}
```

## Security groups and rules

Security groups are a set of security grou rules that apply IP filter rules to instances. Here there a few defined to allow ssh traffic to the bastion as well as HTTP and HTTPS traffic the webserver.

### Create the security groups

Three are being created

- Create security group `public-ssh` for the ssh rule
- Create security group `public-icmp` for the icmp rule
- Create security group `public-web` for the http and https rules

```
## Create security group for public ssh
resource "openstack_networking_secgroup_v2" "public-ssh" {
  name = "public-ssh"
}

## Create security group for public icmp
resource "openstack_networking_secgroup_v2" "public-icmp" {
  name = "public-icmp"
}

## Create security group for public web
resource "openstack_networking_secgroup_v2" "public-web" {
  name = "public-web"
}
```

### Create the security group rules

- Create security group rule `public-ssh` and associate it with `public-ssh` security group
- Create security group rule `public-icmp` and associate it with `public-icmp` security group
- Create security group rule `public-http` and associate it with `public-web` security group
- Create security group rule `public-https` and associate it with `public-web` security group

```
## Create security group rule for public ssh
resource "openstack_networking_secgroup_rule_v2" "public-ssh" {
  direction         = "ingress"
  ethertype         = "IPv4"
  security_group_id = openstack_networking_secgroup_v2.public-ssh.id
  protocol          = "tcp"
  port_range_min    = "22"
  port_range_max    = "22"
  remote_ip_prefix  = "0.0.0.0/0"
}

## Create security group rule for public icmp
resource "openstack_networking_secgroup_rule_v2" "public-icmp" {
  direction         = "ingress"
  ethertype         = "IPv4"
  security_group_id = openstack_networking_secgroup_v2.public-icmp.id
  protocol          = "icmp"
  remote_ip_prefix  = "0.0.0.0/0"
}

## Create security group rule for public http
resource "openstack_networking_secgroup_rule_v2" "public-http" {
  direction         = "ingress"
  ethertype         = "IPv4"
  security_group_id = openstack_networking_secgroup_v2.public-web.id
  protocol          = "tcp"
  port_range_min    = "80"
  port_range_max    = "80"
  remote_ip_prefix  = "0.0.0.0/0"
}

## Create security group rule for public https
resource "openstack_networking_secgroup_rule_v2" "public-https" {
  direction         = "ingress"
  ethertype         = "IPv4"
  security_group_id = openstack_networking_secgroup_v2.public-web.id
  protocol          = "tcp"
  port_range_min    = "443"
  port_range_max    = "443"
  remote_ip_prefix  = "0.0.0.0/0"
}
```

## The compute instances

Now with the network and security group resources setup compute resources can be defined. For each instance a `openstack_compute_instance_v2` resource is defined as well as a `openstack_networking_port_v2` resource to associate the instance with the subnet and security groups. In addition floating ip address will be defined for external access. An `openstack_compute_keypair_v2` resource containing the public key will also be created allowing ssh key based authentication to the instances.

### Adding public key

A public key needs to be added to openstack-flex for ssh key authentication. The default is `~/.ssh/id_rsa.pub` as defined earlier in the variables section.

- Create keypair `public-key`

```
resource "openstack_compute_keypair_v2" "public-key" {
  name = "public-key"
  public_key = file(var.ssh-public-key-path)
}
```

### Bastion server

- Create network port `bastion` and associate the network `internal-network` as well as security groups `public-ssh` and `public-icmp`
- Create the bastion instance `bastion` and associate the network port `bastion` and ssh key `public-key`
- Create floating ip `bastion` and associate it with with networking port `bastion`

```
## Create network port for bastion server
resource "openstack_networking_port_v2" "bastion" {
  name           = "bastion"
  network_id     = openstack_networking_network_v2.internal-network.id
  admin_state_up = "true"

  # Add sucurity groups for public-ssh and public-icmp
  security_group_ids = [openstack_networking_secgroup_v2.public-ssh.id, openstack_networking_secgroup_v2.public-icmp.id]
  fixed_ip {
    subnet_id = openstack_networking_subnet_v2.internal-subnet.id
  }
}

## Create bastion instance
resource "openstack_compute_instance_v2" "bastion" {
  name        = "bastion-server.internal"
  image_name  = "Ubuntu-24.04"
  flavor_name = "gp.0.2.4"
  key_pair    = openstack_compute_keypair_v2.public-key.name
  network {
    port = openstack_networking_port_v2.bastion.id
  }
  metadata = {
    role = "bastion"
  }
}

## Create floating ip for bastion server
resource "openstack_networking_floatingip_v2" "bastion" {
  pool    = "PUBLICNET"
  port_id = openstack_networking_port_v2.bastion.id
}
```
### Web server

- Create network port `webserver` and associate the network `internal-network` as well as security groups `public-web`
- Create the web server instance `webserver` and associate the network port `webserver` and ssh key `public-key`
- Create floating ip `webserver` and associate it with with networking port `webserver`

```
## Create network port for bastion server
resource "openstack_networking_port_v2" "webserver" {
  name           = "webserver"
  network_id     = openstack_networking_network_v2.internal-network.id
  admin_state_up = "true"

  # Add sucurity groups for public-ssh and public-icmp
  security_group_ids = [openstack_networking_secgroup_v2.public-web.id]
  fixed_ip {
    subnet_id = openstack_networking_subnet_v2.internal-subnet.id
  }
}

## Create webserver instance
resource "openstack_compute_instance_v2" "webserver" {
  name        = "webserver.internal"
  image_name  = "Ubuntu-24.04"
  flavor_name = "gp.0.2.4"
  key_pair    = openstack_compute_keypair_v2.public-key.name
  network {
    port = openstack_networking_port_v2.webserver.id
  }
  metadata = {
    role = "webserver"
  }
}

## Create floating ip for webserver
resource "openstack_networking_floatingip_v2" "webserver" {
  pool    = "PUBLICNET"
  port_id = openstack_networking_port_v2.webserver.id
}
```

### Database server

- Create network port `database` and associate the network `internal-network`
- Create the database server instance `database` and associate the network port `database` and ssh key `public-key`

```
## Create network port for database
resource "openstack_networking_port_v2" "database" {
  name           = "database"
  network_id     = openstack_networking_network_v2.internal-network.id
  admin_state_up = "true"

  # Add sucurity groups for public-ssh and public-icmp
  fixed_ip {
    subnet_id = openstack_networking_subnet_v2.internal-subnet.id
  }
}

## Create database instance
resource "openstack_compute_instance_v2" "database" {
  name        = "database.internal"
  image_name  = "Ubuntu-24.04"
  flavor_name = "gp.0.2.4"
  key_pair    = openstack_compute_keypair_v2.public-key.name
  network {
    port = openstack_networking_port_v2.database.id
  }
  metadata = {
    role = "database"
  }
}
```

## Outputs

Outputs in OpenTofu allow for displaying information. These will be used to display the networking information so that the environment can now be accessed. The outputs in this case will be the floating ip address as well as the internal address of the webserver and database server.

```
output "bastion-floating-ip" {
  value = openstack_networking_floatingip_v2.bastion.address
}

output "webserver-floating-ip" {
  value = openstack_networking_floatingip_v2.webserver.address
}

output "webserver-internal-ip" {
  value = openstack_compute_instance_v2.webserver.access_ip_v4
}

output "database-internal-ip" {
  value = openstack_compute_instance_v2.database.access_ip_v4
}
```

## Apply the defined resources

With the `main.tf` file completed it is time to initialize tofu and apply the manifest.

### Initialize OpenTofu
Before you can `apply` the manifest you must first initialize the working directory. This will setup local data and downlad the openstack provider. Using the `upgrade` option is safe as well as running the init multiple time.

```bash
tofu init -upgrade
```

#### Output
```
Initializing the backend...

Initializing provider plugins...
- Finding terraform-provider-openstack/openstack versions matching "~> 3.0.0"...
- Installing terraform-provider-openstack/openstack v3.0.0...
- Installed terraform-provider-openstack/openstack v3.0.0 (signed, key ID 4F80527A391BEFD2)

Providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://opentofu.org/docs/cli/plugins/signing/

OpenTofu has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that OpenTofu can guarantee to make the same selections by default when
you run "tofu init" in the future.

OpenTofu has been successfully initialized!

You may now begin working with OpenTofu. Try running "tofu plan" to see
any changes that are required for your infrastructure. All OpenTofu commands
should now work.

If you ever set or change modules or backend configuration for OpenTofu,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

### Apply the manifest

```bash
tofu apply
```

If everything is setup correctly the changes that will be made will be presented to you with a prompt to apply. Typing yes will cause OpenTofu to run against the openstack endpoint and create your environment.

#### Output

```
data.openstack_networking_network_v2.external-network: Reading...
data.openstack_networking_network_v2.external-network: Read complete after 1s [id=723f8fa2-dbf7-4cec-8d5f-017e62c12f79]

OpenTofu used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

OpenTofu will perform the following actions:

  # openstack_compute_instance_v2.bastion will be created
  + resource "openstack_compute_instance_v2" "bastion" {
      + access_ip_v4        = (known after apply)
      + access_ip_v6        = (known after apply)
      + all_metadata        = (known after apply)
      + all_tags            = (known after apply)
      + availability_zone   = (known after apply)
      + created             = (known after apply)
      + flavor_id           = (known after apply)
      + flavor_name         = "gp.0.2.4"
      + force_delete        = false
      + id                  = (known after apply)
      + image_id            = (known after apply)
      + image_name          = "Ubuntu-24.04"
      + key_pair            = "public-key"
      + metadata            = {
          + "role" = "bastion"
        }
      + name                = "bastion-server.internal"
      + power_state         = "active"
      + region              = (known after apply)
      + security_groups     = (known after apply)
      + stop_before_destroy = false
      + updated             = (known after apply)

      + network {
          + access_network = false
          + fixed_ip_v4    = (known after apply)
          + fixed_ip_v6    = (known after apply)
          + mac            = (known after apply)
          + name           = (known after apply)
          + port           = (known after apply)
          + uuid           = (known after apply)
        }
    }

  # openstack_compute_instance_v2.database will be created
  + resource "openstack_compute_instance_v2" "database" {
      + access_ip_v4        = (known after apply)
      + access_ip_v6        = (known after apply)
      + all_metadata        = (known after apply)
      + all_tags            = (known after apply)
      + availability_zone   = (known after apply)
      + created             = (known after apply)
      + flavor_id           = (known after apply)
      + flavor_name         = "gp.0.2.4"
      + force_delete        = false
      + id                  = (known after apply)
      + image_id            = (known after apply)
      + image_name          = "Ubuntu-24.04"
      + key_pair            = "public-key"
      + metadata            = {
          + "role" = "database"
        }
      + name                = "database.internal"
      + power_state         = "active"
      + region              = (known after apply)
      + security_groups     = (known after apply)
      + stop_before_destroy = false
      + updated             = (known after apply)

      + network {
          + access_network = false
          + fixed_ip_v4    = (known after apply)
          + fixed_ip_v6    = (known after apply)
          + mac            = (known after apply)
          + name           = (known after apply)
          + port           = (known after apply)
          + uuid           = (known after apply)
        }
    }

  # openstack_compute_instance_v2.webserver will be created
  + resource "openstack_compute_instance_v2" "webserver" {
      + access_ip_v4        = (known after apply)
      + access_ip_v6        = (known after apply)
      + all_metadata        = (known after apply)
      + all_tags            = (known after apply)
      + availability_zone   = (known after apply)
      + created             = (known after apply)
      + flavor_id           = (known after apply)
      + flavor_name         = "gp.0.2.4"
      + force_delete        = false
      + id                  = (known after apply)
      + image_id            = (known after apply)
      + image_name          = "Ubuntu-24.04"
      + key_pair            = "public-key"
      + metadata            = {
          + "role" = "webserver"
        }
      + name                = "webserver.internal"
      + power_state         = "active"
      + region              = (known after apply)
      + security_groups     = (known after apply)
      + stop_before_destroy = false
      + updated             = (known after apply)

      + network {
          + access_network = false
          + fixed_ip_v4    = (known after apply)
          + fixed_ip_v6    = (known after apply)
          + mac            = (known after apply)
          + name           = (known after apply)
          + port           = (known after apply)
          + uuid           = (known after apply)
        }
    }

  # openstack_compute_keypair_v2.public-key will be created
  + resource "openstack_compute_keypair_v2" "public-key" {
      + fingerprint = (known after apply)
      + id          = (known after apply)
      + name        = "public-key"
      + private_key = (sensitive value)
      + public_key  = <<-EOT
            ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDBnanLPVimKowjVYKoSV38IOej8yHlTOn2g/vr8e3RhGQBFq4XuWiEIYJhbyRf+3RW6oy8dw91auotQCMJuwL/+QTMCt9RDGT1ygWKHUb+0NQ3METwQKBiiD7Ml1jifJmJRnxQ5Sha86Ha+cNwsvj3krhcyNNi9aZfMIok0C3q3+mGIQ9EyyccSlylGtE3X3r/pyzRkMf0LH3vMP61VNpB4oeEQQ6+hHPYH4d0yBiTvU73dumx+gr7u3/DHIq4C62Uk+LuOpMidNjlvlZVHXjaLmEn0re8U+awP1zfU1FOJIH0yqpu4glJbN325kMunJ/1mWcGknoV0srsBBHyFNTz cblument@localhost.localdomain
        EOT
      + region      = (known after apply)
      + user_id     = (known after apply)
    }

  # openstack_networking_floatingip_v2.bastion will be created
  + resource "openstack_networking_floatingip_v2" "bastion" {
      + address    = (known after apply)
      + all_tags   = (known after apply)
      + dns_domain = (known after apply)
      + dns_name   = (known after apply)
      + fixed_ip   = (known after apply)
      + id         = (known after apply)
      + pool       = "PUBLICNET"
      + port_id    = (known after apply)
      + region     = (known after apply)
      + subnet_id  = (known after apply)
      + tenant_id  = (known after apply)
    }

  # openstack_networking_floatingip_v2.webserver will be created
  + resource "openstack_networking_floatingip_v2" "webserver" {
      + address    = (known after apply)
      + all_tags   = (known after apply)
      + dns_domain = (known after apply)
      + dns_name   = (known after apply)
      + fixed_ip   = (known after apply)
      + id         = (known after apply)
      + pool       = "PUBLICNET"
      + port_id    = (known after apply)
      + region     = (known after apply)
      + subnet_id  = (known after apply)
      + tenant_id  = (known after apply)
    }

  # openstack_networking_network_v2.internal-network will be created
  + resource "openstack_networking_network_v2" "internal-network" {
      + admin_state_up          = true
      + all_tags                = (known after apply)
      + availability_zone_hints = (known after apply)
      + dns_domain              = (known after apply)
      + external                = false
      + id                      = (known after apply)
      + mtu                     = (known after apply)
      + name                    = "internal-network"
      + port_security_enabled   = true
      + qos_policy_id           = (known after apply)
      + region                  = (known after apply)
      + shared                  = (known after apply)
      + tenant_id               = (known after apply)
      + transparent_vlan        = (known after apply)
    }

  # openstack_networking_port_v2.bastion will be created
  + resource "openstack_networking_port_v2" "bastion" {
      + admin_state_up         = true
      + all_fixed_ips          = (known after apply)
      + all_security_group_ids = (known after apply)
      + all_tags               = (known after apply)
      + device_id              = (known after apply)
      + device_owner           = (known after apply)
      + dns_assignment         = (known after apply)
      + dns_name               = (known after apply)
      + id                     = (known after apply)
      + mac_address            = (known after apply)
      + name                   = "bastion"
      + network_id             = (known after apply)
      + port_security_enabled  = (known after apply)
      + qos_policy_id          = (known after apply)
      + region                 = (known after apply)
      + security_group_ids     = (known after apply)
      + tenant_id              = (known after apply)

      + fixed_ip {
          + subnet_id = (known after apply)
        }
    }

  # openstack_networking_port_v2.database will be created
  + resource "openstack_networking_port_v2" "database" {
      + admin_state_up         = true
      + all_fixed_ips          = (known after apply)
      + all_security_group_ids = (known after apply)
      + all_tags               = (known after apply)
      + device_id              = (known after apply)
      + device_owner           = (known after apply)
      + dns_assignment         = (known after apply)
      + dns_name               = (known after apply)
      + id                     = (known after apply)
      + mac_address            = (known after apply)
      + name                   = "database"
      + network_id             = (known after apply)
      + port_security_enabled  = (known after apply)
      + qos_policy_id          = (known after apply)
      + region                 = (known after apply)
      + tenant_id              = (known after apply)

      + fixed_ip {
          + subnet_id = (known after apply)
        }
    }

  # openstack_networking_port_v2.webserver will be created
  + resource "openstack_networking_port_v2" "webserver" {
      + admin_state_up         = true
      + all_fixed_ips          = (known after apply)
      + all_security_group_ids = (known after apply)
      + all_tags               = (known after apply)
      + device_id              = (known after apply)
      + device_owner           = (known after apply)
      + dns_assignment         = (known after apply)
      + dns_name               = (known after apply)
      + id                     = (known after apply)
      + mac_address            = (known after apply)
      + name                   = "webserver"
      + network_id             = (known after apply)
      + port_security_enabled  = (known after apply)
      + qos_policy_id          = (known after apply)
      + region                 = (known after apply)
      + security_group_ids     = (known after apply)
      + tenant_id              = (known after apply)

      + fixed_ip {
          + subnet_id = (known after apply)
        }
    }

  # openstack_networking_router_interface_v2.internal-router-interface will be created
  + resource "openstack_networking_router_interface_v2" "internal-router-interface" {
      + force_destroy = false
      + id            = (known after apply)
      + port_id       = (known after apply)
      + region        = (known after apply)
      + router_id     = (known after apply)
      + subnet_id     = (known after apply)
    }

  # openstack_networking_router_v2.external-router will be created
  + resource "openstack_networking_router_v2" "external-router" {
      + admin_state_up          = true
      + all_tags                = (known after apply)
      + availability_zone_hints = (known after apply)
      + distributed             = (known after apply)
      + enable_snat             = (known after apply)
      + external_network_id     = "723f8fa2-dbf7-4cec-8d5f-017e62c12f79"
      + id                      = (known after apply)
      + name                    = "external-router"
      + region                  = (known after apply)
      + tenant_id               = (known after apply)
    }

  # openstack_networking_secgroup_rule_v2.public-http will be created
  + resource "openstack_networking_secgroup_rule_v2" "public-http" {
      + direction         = "ingress"
      + ethertype         = "IPv4"
      + id                = (known after apply)
      + port_range_max    = 80
      + port_range_min    = 80
      + protocol          = "tcp"
      + region            = (known after apply)
      + remote_group_id   = (known after apply)
      + remote_ip_prefix  = "0.0.0.0/0"
      + security_group_id = (known after apply)
      + tenant_id         = (known after apply)
    }

  # openstack_networking_secgroup_rule_v2.public-https will be created
  + resource "openstack_networking_secgroup_rule_v2" "public-https" {
      + direction         = "ingress"
      + ethertype         = "IPv4"
      + id                = (known after apply)
      + port_range_max    = 443
      + port_range_min    = 443
      + protocol          = "tcp"
      + region            = (known after apply)
      + remote_group_id   = (known after apply)
      + remote_ip_prefix  = "0.0.0.0/0"
      + security_group_id = (known after apply)
      + tenant_id         = (known after apply)
    }

  # openstack_networking_secgroup_rule_v2.public-icmp will be created
  + resource "openstack_networking_secgroup_rule_v2" "public-icmp" {
      + direction         = "ingress"
      + ethertype         = "IPv4"
      + id                = (known after apply)
      + protocol          = "icmp"
      + region            = (known after apply)
      + remote_group_id   = (known after apply)
      + remote_ip_prefix  = "0.0.0.0/0"
      + security_group_id = (known after apply)
      + tenant_id         = (known after apply)
    }

  # openstack_networking_secgroup_rule_v2.public-ssh will be created
  + resource "openstack_networking_secgroup_rule_v2" "public-ssh" {
      + direction         = "ingress"
      + ethertype         = "IPv4"
      + id                = (known after apply)
      + port_range_max    = 22
      + port_range_min    = 22
      + protocol          = "tcp"
      + region            = (known after apply)
      + remote_group_id   = (known after apply)
      + remote_ip_prefix  = "0.0.0.0/0"
      + security_group_id = (known after apply)
      + tenant_id         = (known after apply)
    }

  # openstack_networking_secgroup_v2.public-icmp will be created
  + resource "openstack_networking_secgroup_v2" "public-icmp" {
      + all_tags    = (known after apply)
      + description = (known after apply)
      + id          = (known after apply)
      + name        = "public-icmp"
      + region      = (known after apply)
      + stateful    = (known after apply)
      + tenant_id   = (known after apply)
    }

  # openstack_networking_secgroup_v2.public-ssh will be created
  + resource "openstack_networking_secgroup_v2" "public-ssh" {
      + all_tags    = (known after apply)
      + description = (known after apply)
      + id          = (known after apply)
      + name        = "public-ssh"
      + region      = (known after apply)
      + stateful    = (known after apply)
      + tenant_id   = (known after apply)
    }

  # openstack_networking_secgroup_v2.public-web will be created
  + resource "openstack_networking_secgroup_v2" "public-web" {
      + all_tags    = (known after apply)
      + description = (known after apply)
      + id          = (known after apply)
      + name        = "public-web"
      + region      = (known after apply)
      + stateful    = (known after apply)
      + tenant_id   = (known after apply)
    }

  # openstack_networking_subnet_v2.internal-subnet will be created
  + resource "openstack_networking_subnet_v2" "internal-subnet" {
      + all_tags          = (known after apply)
      + cidr              = "192.168.50.0/24"
      + enable_dhcp       = true
      + gateway_ip        = (known after apply)
      + id                = (known after apply)
      + ip_version        = 4
      + ipv6_address_mode = (known after apply)
      + ipv6_ra_mode      = (known after apply)
      + name              = "internal-subnet"
      + network_id        = (known after apply)
      + no_gateway        = false
      + region            = (known after apply)
      + service_types     = (known after apply)
      + tenant_id         = (known after apply)

      + allocation_pool {
          + end   = "192.168.50.254"
          + start = "192.168.50.10"
        }
    }

Plan: 20 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bastion-floating-ip   = (known after apply)
  + database-internal-ip  = (known after apply)
  + webserver-floating-ip = (known after apply)
  + webserver-internal-ip = (known after apply)

Do you want to perform these actions?
  OpenTofu will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

openstack_compute_keypair_v2.public-key: Creating...
openstack_networking_secgroup_v2.public-icmp: Creating...
openstack_networking_secgroup_v2.public-ssh: Creating...
openstack_networking_secgroup_v2.public-web: Creating...
openstack_networking_network_v2.internal-network: Creating...
openstack_networking_router_v2.external-router: Creating...
openstack_compute_keypair_v2.public-key: Creation complete after 2s [id=public-key]
openstack_networking_secgroup_v2.public-ssh: Creation complete after 4s [id=7853d33d-56ac-463b-b3c2-035705e8495c]
openstack_networking_secgroup_rule_v2.public-ssh: Creating...
openstack_networking_secgroup_v2.public-web: Creation complete after 5s [id=e13ed13f-fb5f-479a-ae12-62dfc1abd679]
openstack_networking_secgroup_rule_v2.public-http: Creating...
openstack_networking_secgroup_rule_v2.public-https: Creating...
openstack_networking_secgroup_v2.public-icmp: Creation complete after 5s [id=67d38c5b-5b78-4c4f-b9d3-59d2f790460f]
openstack_networking_secgroup_rule_v2.public-icmp: Creating...
openstack_networking_secgroup_rule_v2.public-ssh: Creation complete after 3s [id=3892d932-923f-48dd-b673-7ac17ce1aa64]
openstack_networking_secgroup_rule_v2.public-http: Creation complete after 2s [id=4b48f11a-060c-400d-99d8-f0f3996373ea]
openstack_networking_secgroup_rule_v2.public-icmp: Creation complete after 4s [id=8bf37903-e938-41d9-abbf-f3143bc9f331]
openstack_networking_network_v2.internal-network: Still creating... [10s elapsed]
openstack_networking_router_v2.external-router: Still creating... [10s elapsed]
openstack_networking_secgroup_rule_v2.public-https: Creation complete after 8s [id=d08b6ee7-5868-4020-82e1-9f88c5dd094b]
openstack_networking_network_v2.internal-network: Creation complete after 15s [id=1a9e292a-4e59-4524-9612-5a66f80f3e3c]
openstack_networking_subnet_v2.internal-subnet: Creating...
openstack_networking_router_v2.external-router: Still creating... [20s elapsed]
openstack_networking_subnet_v2.internal-subnet: Creation complete after 7s [id=c48b8733-f363-48ca-b96a-8bbf91f9a57a]
openstack_networking_port_v2.database: Creating...
openstack_networking_port_v2.webserver: Creating...
openstack_networking_port_v2.bastion: Creating...
openstack_networking_router_v2.external-router: Creation complete after 24s [id=c5e490f5-404a-4c07-abcd-0a5caa08c9ca]
openstack_networking_router_interface_v2.internal-router-interface: Creating...
openstack_networking_port_v2.database: Creation complete after 7s [id=c951246d-3d44-4c4b-a9d1-30c6629f5fe4]
openstack_compute_instance_v2.database: Creating...
openstack_networking_port_v2.webserver: Creation complete after 7s [id=9456c5e6-2aa2-4b6b-8191-c91ffa16b124]
openstack_networking_floatingip_v2.webserver: Creating...
openstack_compute_instance_v2.webserver: Creating...
openstack_networking_port_v2.bastion: Creation complete after 8s [id=40a5722b-bd26-4789-a2e9-d07a9dc335f4]
openstack_networking_floatingip_v2.bastion: Creating...
openstack_compute_instance_v2.bastion: Creating...
openstack_networking_router_interface_v2.internal-router-interface: Creation complete after 10s [id=570f1fb4-2fee-4c8e-87d1-2057a93a91f0]
openstack_networking_floatingip_v2.bastion: Creation complete after 8s [id=b39ffdaf-330b-4e76-a8cb-5f591feff184]
openstack_networking_floatingip_v2.webserver: Creation complete after 10s [id=bbf419b5-4a4d-4845-916a-64ddf265ce5c]
openstack_compute_instance_v2.database: Still creating... [10s elapsed]
openstack_compute_instance_v2.webserver: Still creating... [10s elapsed]
openstack_compute_instance_v2.bastion: Still creating... [10s elapsed]
openstack_compute_instance_v2.database: Creation complete after 13s [id=da4c4660-3afd-466b-be86-5b5af48c3b91]
openstack_compute_instance_v2.webserver: Creation complete after 14s [id=ad4660f7-91de-4b79-92a9-05dc775795b6]
openstack_compute_instance_v2.bastion: Creation complete after 13s [id=1800a88f-b6b7-49d3-9809-e58822d6d1bb]

Apply complete! Resources: 20 added, 0 changed, 0 destroyed.

Outputs:

bastion-floating-ip = "REDACTED"
database-internal-ip = "192.168.50.103"
webserver-floating-ip = "REDACTED"
webserver-internal-ip = "192.168.50.86"
```
