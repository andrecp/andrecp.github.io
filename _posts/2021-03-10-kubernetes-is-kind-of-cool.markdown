---
layout: post
title:  "Kubernetes is kind of cool"
date:   2021-03-10 08:55:38 -0700
categories: devops kubernetes
---

Having helped a couple of companies in the devops space I feel very comfortable with the tech stack I am the most familiar with (shocking):
* Consul
* Ansible
* Some monitoring tool hooked into Consul (Prometheus, Sensu and Zabbix)
* The ELK stack for logging and metrics
* Systemd / Docker
* Jenkins
* VMs
* NGINX
* Bash

When *properly* configured it almost feels like magic! You do a git push and you have your service running somewhere with a DNS, load balancing, monitoring and centralized logging. It also scales really well to multiple datacenter and developers! You can have a rough idea on how it looks like in the [PyCon presentation](https://www.youtube.com/watch?v=HuV0_AbDgYk) I gave.


It's a REST API written in python3.8 and the only dependencies so far are for [passlib](https://passlib.readthedocs.io/en/stable/), [jwt](https://pyjwt.readthedocs.io/en/latest/) and [tornado](http://tornadoweb.org/en/stable/web.html). It uses [sqlite](https://www.sqlite.org/index.html) as the database.

However, It's also a lot of moving parts! You need developers to learn these 8-9 technologies to a certain degree and for them to have a mental model on how they connect together. Even if you give them abstractions, they're usually not enough for every use case your business has.

Suddenly your Ansible scripts are full of if statements, or you have ~200 galaxy roles to deal with every use case that arises, or you have developers doing their own thing because what you created is too complex for them to grasp! Developers very rarely want (or should) care too much about the infrastructure. They operate on a level above it.

And what happens when a developer changes companies? They have to learn how that particular company implemented the devops workflows again! And when you change companies? How to take the existing custom infrastructure from their state to a better state.

How can Kubernetes help to solve this problem?

Well, still early days for me, but, I can totally see how Kubernetes is an opinionated implementation of some devops workflows! 

* Do you need some docker containers running? Pod
* Do you need filebeats / prometheus running on every machine? Daemonset
* Do you need DNS / service discovery? CoreDNS
* Do you need to get your service behind a load balancer? Ingress
* Strategy for rolling out newer versions of your service ? Deployments / * StatefulSet
* Storage? PV/PVC/Storage Class 

Kubernetes seems to implement all the patterns that you would have to implement & glue yourself together anyways! All with a very nice unified API to your infrastructure and as code.

Developers can now learn a single implementation of devops (or not learn it at all) and go back to the heroku model, where they develop with the [12factor app](https://12factor.net) patterns in mind and get their infrastructure + code to prod with a git push!

Devops engineers can now learn a single implementation of devops to deploy a platform for developers (logging, monitoring, ci/cd, etc.) on prems, cloud or what have you.

I am sure there will be a lot of hurdles and weird errors to troubleshoot in a k8s cluster, but so far, I am enjoying all the yaml, is kind of cool.
