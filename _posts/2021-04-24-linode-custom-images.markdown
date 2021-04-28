---
layout: post
title:  "Creating a custom Linode with Terraform and Packer"
date:   2021-04-24 08:55:38 -0700
tags: devops cloud
---

I struggled a little bit trying to create a Linode VM with a custom image using Packer and Terraform, thought I would share how I did here to help the next person! My main issue was that the disk space of the VM created from a custom image was ~ 5 GB instead of the advertised ~160 GB for the plan I was using.

Requirements:
* [Install Packer](https://learn.hashicorp.com/tutorials/packer/getting-started-install)
* [Install Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
    * You can optionally sign up for terraform cloud too
    * Their free account supports up to 5 users and allows you to store the state of TF remotely, as well as the variables per workspace
* [Sign up for Linode](https://www.googleadservices.com/pagead/aclk?sa=L&ai=CCcr5sGaDYMfYJ6P6tOUP5di8-A281cCjYtqdlK69DKDWuZRHCAAQAiC5VCgEYP2Y-4DMA6AB9ZyR_wPIAQHIA9ggqgRCT9BA97YlfNKfVNWGucyNGth_on6pi2Bng27G1BbqRKewCMgcNmNQvVtE6blzQpYWpVVr2kBElFMqf-hFpZjw19qFwASrsNnclgOABZBOoAZRgAfz4m6QBwGoB6a-G6gHuZqxAqgH8NkbqAfy2RuoB_PRG6gH7tIbqAfK3BuwCAHSCAYQASCkgQKaCSxodHRwczovL3d3dy5saW5vZGUuY29tL2xwL2JyYW5kLWZyZWUtY3JlZGl0L7EJcjnYcOk8rFS5CXI52HDpPKxU-AkBmAsBuAwB0BUBgBcBkhcGEgQIARAD&ae=2&ved=2ahUKEwjlya6t0JXwAhXYpJ4KHSg8BesQ0Qx6BAgGEAE&dct=1&dblrd=1&val=Ggh9IulWtNYZgyABKAAwk47t_vXT8-dhOLDNjYQGQLDNjYQG&sig=AOD64_0BAlZg7bs80qnxq5NQTxDuR3zLsw&adurl=https://www.linode.com/lp/brand-free-credit/%3Futm_source%3Dgoogle%26utm_medium%3Dcpc%26utm_campaign%3D11178784465_109179197483%26utm_term%3Dg_kwd-19101805344_b_%252Blinode%26utm_content%3D466889241808%26locationid%3D9001551%26device%3Dc_c) 
    * As of right now they're giving $100 credits to be used in two months

Note that this might incur some small costs if you're not using the trial credits.

# Packer

Packer allows you to create a template for a VM by executing repeatable codified steps, in my case I'm creating a new template in Linode based on the opensuse15.2 (Leap) image that Linode already offers.

[Official docs](https://www.packer.io/docs/builders/linode)

My template has sshd login with password disabled, a different port for ssh and docker installed. 

```t
{
    "variables": {
        "linode_token": "{{env `LINODE_TOKEN`}}"
    },

    "builders": [
        {
            "type": "linode",
            "linode_token": "{{user `linode_token`}}",
            "image": "linode/opensuse15.2",
            "region": "us-west",
            "instance_type": "g6-standard-4",
            "image_label": "template-{{timestamp}}",
            "image_description": "a template",
            "ssh_username": "root"
        }
    ],

    "provisioners": [
        {
            "type": "shell",
            "inline": [
                "zypper refresh",
                "zypper update -y",
                "zypper install -y docker python3-docker-compose",
                "systemctl enable docker"
            ]
        },
        {
            "type": "file",
            "source": "sshd_config",
            "destination": "/etc/ssh/sshd_config"
        }
    ]
}
```

Nothing too crazy. Here is my sshd_config if you're curious  and want to criticize (put on the same folder)

```t
Port 30001
PermitRootLogin without-password
AuthorizedKeysFile      .ssh/authorized_keys
PasswordAuthentication no
ChallengeResponseAuthentication no
X11Forwarding no
Subsystem       sftp    /usr/lib/ssh/sftp-server
AcceptEnv LANG LC_CTYPE LC_NUMERIC LC_TIME LC_COLLATE LC_MONETARY LC_MESSAGES
AcceptEnv LC_PAPER LC_NAME LC_ADDRESS LC_TELEPHONE LC_MEASUREMENT
AcceptEnv LC_IDENTIFICATION LC_ALL
```

As you can see, port 30001 and disabled password auth.

Great ! Now we can create a new image template with Packer.

```bash
> export LINODE_TOKEN=$YOUR_LINODE_TOKEN # Create from the web UI
> packer validate packer.json
> packer build packer.json # Packer will take a bit to run and give you back a string, like, private/11860121
```

*private/11860121* is your new image id! 

Packer resizes the Linode Image before making it a template, which makes sense as you want a very small template image to not occupy too much storage. And that's where the trouble started...

# Terraform

Terraform is another tool from hashicorp and it helps you to spin infrastructure on cloud providers! The [official docs for Linode](https://registry.terraform.io/providers/linode/linode/latest/docs) mention the disk settings, but, it doesn't mention that when you use private image you need to use them and configure the disk, otherwise, you lose the remaining space and your VM only has the ~5 GB of the template to use.

This is my main.tf

```t
terraform {
  required_providers {
    linode = {
      source = "linode/linode"
    }
  }

  # Remove this bit if not using terraform cloud
  backend "remote" {
    organization = "myterraformcloud99"

    workspaces {
      name = "linode-prod"
    }
  }
}

variable "linode_token" {
  type = string
}

variable "linode_ssh_pub_key" {
  type = string
}

variable "root_password" {
  type = string
}

provider "linode" {
  token = var.linode_token
}

resource "linode_instance" "srv_01" {
  label             = "server-01"
  region            = "us-west"
  type              = "g6-standard-4"
  boot_config_label = "boot_config"

  disk {
    image  = "private/11860121" # Generated with Packer
    label = "boot"
    size = "159488"  # The promised 160 GB - 512 MB for swap
    filesystem = "ext4"
    root_pass = var.root_password
    authorized_keys = [var.linode_ssh_pub_key]
  }

  disk {
    label = "swap"
    filesystem = "swap"
    size = "512"
  }

  config {
    label = "boot_config"
    kernel = "linode/latest-64bit"
    root_device = "/dev/sda"
    devices {
      sda {
        disk_label = "boot"
      }
      sdb {
        disk_label = "swap"
      }
    }
  }
}

output "public_ip" {
  value = linode_instance.srv_01.ip_address
}
```

As you can see, there's quite a few things there compared to the minimum example in the official Terraform docs! And it seems that you absolutely need it to make it work for custom images. 

I'm using terraform cloud and storing the variables as secrets in my workspace, regardless, you will need to make sure you have variables for:

1. Your ssh key (which you should have generated before hand and uploaded to your account, you will get locked out of your VM if you don't add them at creation time)
2. Your linode token
3. Your root account password (you disabled ssh login with password anyways, but, it's required by Linode. Use a strong password)

```bash
> terraform init

# Login if you using cloud and storing your variables there, otherwise, you have to pass the variables via CLI or via a vars file.
> terraform login

> terraform plan
> terraform apply -auto-approve # Don't use -auto-approve if the plan step looks nonsense!

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

public_ip = "xx.xx.xx.xx"
```

Aaand that's it!

We can connect to the public IP from the terraform run like this:

```bash
ssh -i ~/.ssh/linode root@xx.xx.xx.xx -p 30001
```

Hopefully this helps someone! I scratched my head quite a bit to understand where my disk space was going, and after that, to make it boot correctly.
