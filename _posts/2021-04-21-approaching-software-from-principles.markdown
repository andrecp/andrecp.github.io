---
layout: post
title:  "Approaching software problems from first principles"
date:   2021-04-21 08:55:38 -0700
categories: software
---

One thing that I've noticed across my career is how many of us developers sometimes get caught into the details about a particular software tool when trying to troubleshoot an issue, rather than truly understanding what the software is trying to achieve and what might be preventing it from in a systematic level.

As an example, if a REST API is slow, folks might spend a lot of time trying to understand the code, the business logic, or the framework the app is using. While that is valuable and there's a space for it, I feel that approaching from the system point of view can be much more valuable as a first step. For example:

* Is the machine OK?
    * Is the load average in the machine alright?
    * Is the iowait in the machine alright?
    * Is the memory usage alright?
    * sar
    * dmesg
* Is the process OK?
    * Systemd journal
    * Logs
    * ps aux
* Is the network OK?
    * Is it DNS?
    * Is it a load balancer?
* etc.

The beauty of above checks is that it can be applied to *any REST API written in any framework*, in fact, it can be applied to any software application running in a Linux system.

If you truly master the underlying principles of the Operational System and Computers you need to keep much less context in your head. It "scales" your mental bandwidth better, instead of trying to know details about 10 different apps I only need to know Linux when troubleshooting a problem. This is truly valuable in DevOps/Platform space where you might need to have the whole org in your head!

Of course that, when the underlying principles of the system are OK, then it's time to dig deeper into the business logic, framework and what have you.

The opposite of this approach is to start troubleshooting an issue by adding log statements, doing CURL requests, clicking buttons, trying to understand the code, the business logic and etc. Before even knowing if the problem is in the application at all.

This also applies when developing software or stitching systems together! The differences between the different tools on the monitoring, queues, databases, REST frameworks and etc. spaces are less important than understanding their underlying principles. Tools are just tools (do learn a couple of them well!).

I recommend reading what [rachelbythebay](https://rachelbythebay.com/w/) writes and the zines [Julia Evans](https://twitter.com/b0rk) puts out there if you want to understand better the underlying systems!
