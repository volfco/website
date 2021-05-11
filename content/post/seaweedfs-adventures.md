---
title: "Gitall"
date: 2018-09-29T11:36:33+08:00
draft: true
tags: 
  - gitall
  - project
---

# Adventures in Homelab Storage

Homelabs are fun, if you're into that kind of thing. 

Kubernetes & Nomad both support the [Container Storage Interface](https://github.com/container-storage-interface/spec), which provides an easy way to attach presistant storage to your container. It's a 

For a number of reasons, I need a distributed storage system that can scale out for my homelab. There are a few options out there, summed up by https://github.com/okhosting/awesome-storage

The main features I'm looking for are:
1. Erasure Encoding
2. Ability to add single nodes & re-balance data
3. Object Storage (S3 Ideal)

**Ceph** is the "industry standard" which means there is a lot of info out there, but it's also a complex piece of software to understand and deploy. It's a technology I want to explore down the road, but I would want to have a more robust lab environment before progressing.

**LeoFS, XtreemFS, and HekaFS** all seem to be abandoned or hiatus. LeoFS specifically seems to be stable, but development paused. 

**Cortx** by Segate is something I really want to explore, but like Ceph, it looks dense and messy to setup. 
 
**LizardFS** also looks to be in development purgatory. The documentation is broken and out of date, but there seems to be some active development going on. There isn't enough information out there on current & future state to consider moving forward with testing.

**OrangeFS** looks to be semi-active in development, and is releasing stable versions. Looks to be similar to Gluster in that it provides a distributed POSIX filesystem. There is limited information out there, which makes this a fun weekend project. But not right now. 

**GlusterFS** is a system I've used in the past. 

**SeaweedFS** is a very appealing project. Modern implementation of battle tested concepts, with an easy to understand architecture.

**BeeGFS** https://github.com/stackhpc/ansible-role-beegfs

## The Lab
For the 1st phase of this project, I want to get my 3 NUCs to act as part of a SeaweedFS cluster. This will be a triple master deployment, where the master process will run on every node. along side volume servers, and filers. 

Easiest to manage and I don't have to worry about single points of failures. 

- Ubuntu 20.04
- 3x Intel NUCs of varying generations
- i7s, 64G of RAM, and 500G Disk

Each SeaweedFS component is deployed along side a RKE Kubernetes Deployment. 

# SeaweedFS
## The Ansible
https://github.com/bmillemathias/ansible-role-seaweedfs

## Initial Deployment 

I'm doing a three node deployment, which means that each component will be deployed on each node- Master, Volume, and Filer. 

The resulting ansible looked like this 

```yaml

all:
  children:
    seaweedfs:
      children:
        weed_master:
          hosts:
            docker1.lab.volf.co:
            docker2.lab.volf.co:
            docker3.lab.volf.co:
        weed_volume:
          hosts:
            docker1.lab.volf.co:
            docker2.lab.volf.co:
            docker3.lab.volf.co:
        weed_filer:
          hosts:
            docker1.lab.volf.co:
            docker2.lab.volf.co:
            docker3.lab.volf.co:      
      vars:
        domain: lab.volf.co
        weed:
          version: '2.36'
          large_disk: False
          bind: 0.0.0.0
          defaultReplication: "001"
          master:
            port: 9333
            dir: "/opt/seaweedfs/{{ domain }}/master"
        
          volume:
            port: 9334
            dir: "/opt/seaweedfs/{{ domain }}/volume"
            dataCenter: DefaultDataCenter
            rack: DefaultRack
            max_volumes: 1024

          filer:
            port: 9335
            dir: "/opt/seaweedfs/{{ domain }}/filer"
            encryptData: false

```

### Replication Setting
Take special note of the defaultReplication setting. The official seaweedfs docs are not clear on the default setting, but the ansible module used here will default to "000." This means there is no replication configured. 

The way seaweedfs handles replication is at the per-volume level, and you can configure replication to be done across datacenters, racks, or servers. 

**000** means that replicate across **0 Datacenters**, **0 Racks**, and ***0 Servers**. Resulting in no fault tolerance. 

I'm using **001**, which means that there will be 1 replica on another server. Which allows for a single node failure- good enough for my homelab. There is also erasure encoding, but that is something I'll explore later.

## Running 
Let's run Ansible!

## The Master
Assuming that Ansible ran cleanly, you should have a fully working cluster. But let's test this. 

## The Volumes

## The Filer
The Filter is the optional part of this setup, according to the docs. But this is a vital part of the system for our uses. 

## The Issues
The biggest issue I ran smack into is performance. The Fuse driver is slow

# Trying Ceph