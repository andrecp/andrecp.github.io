---
layout: post
title:  "Creating a VM in Digital Ocean with Terraform"
date:   2021-03-14 08:55:38 -0700
tags: devops terraform cloud
---

Today I created a virtual machine in Digital Ocean with Terraform! Had been a while, it's surprising how straight forward things can be now a days in the DevOps space. You can register in Digital Ocean [here](https://cloud.digitalocean.com/registrations/new), create a personal token [here](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2#HowToGenerateaPersonalAccessToken) and generate the ssh keys [here](https://www.digitalocean.com/community/tutorials/how-to-use-ssh-keys-with-digitalocean-droplets).

Really solid docs on [how to install and do a quick start](https://learn.hashicorp.com/tutorials/terraform/install-cli) in Terraform as well as the [Digital Ocean provider](https://registry.terraform.io/providers/digitalocean/digitalocean/latest/docs). Less than 5 minutes and you have your automation for creating infrastructure!

Put the following content in a *main.tf* file after installing terraform

```t
# Define variables
variable "do_token" {}
variable "do_sshkey" {}

# Define dependencies (The digital ocean provider)
terraform {
  required_providers {
    digitalocean = {
      source = "digitalocean/digitalocean"
    }
  }
}

# Give the provider your token
provider "digitalocean" {
  token = var.do_token
}

# Define the VM resource, ubuntu LTS in SF with the cheapest configuration
resource "digitalocean_droplet" "web" {
  image    = "ubuntu-20-04-x64"
  name     = "web-1"
  region   = "sfo3"
  size     = "s-1vcpu-1gb"
  ssh_keys = [var.do_sshkey]
}
```

And do it

```bash
> terraform init
> terraform validate

# Replace xxxxxxx with your personal access token and yyyyyyy with your ssh key id.

# See what terraform will do!
> terraform plan --var do_sshkey=yyyyyyy --var do_token=xxxxxxx

# Create the VM!
> terraform apply --var do_sshkey=yyyyyyy --var do_token=xxxxxxx

# Get the IP
> terraform show | grep "ipv4"

# Connect to it, replace xxxx.xxxx.xxxx.xxxx with your IP.
> ssh root@xxxx.xxxx.xxxx.xxxx 
```

Once you're done playing with with it, delete it so that it doesn't cost you money

```bash
terraform destroy --var do_sshkey=yyyyyyy --var do_token=xxxxxxx
```

You can check the final repo in my github [here](https://github.com/andrecp/devops-fundamentals-to-k8s/tree/main/chapter2-1-setting-up-your-environment)
