---
title: "Exploring Clickhouse Users"
date: 2018-09-29T11:36:33+08:00
draft: true
tags: 
  - Clickhouse
---


There's a big question on how do you sync Let's Encrypt certs between active-passive or active-active load balancers.

In all my searching, I have not found anyone who has presented a way to do this other then copying the certs by hand manually. 

I'm assuming you're running your load balancers on VM or Physical box. This doesn't really work if you're runningn in a containerized envornment, such as Nomad or Kubernetes. There are a bunch of solutions built specifically for them.


I'm using keepalived to make sure my Public IP is pointing at the correct server. It's the common way for load balancers to fail over.

Because we know which server is active, and which is not, we can use this as a "lock" to prevent the passive servers from trying to request updates. 

A cron job runs on the active node that requests updates. Once it does that, it fires off an rsync job to move all the local certificates to other nodes

This way should work in both 