+++
date = 2021-05-13T03:29:38Z
draft = true
tags = []
title = "Building my Cloud. One step at a time."

+++

So. Let's break down the required components needed for a minimal platform.

# Components

## The Hypervisor

Workloads need to run somewhere, and the hypervisor is where they will run. The popular products out there are:

* KVM / QEMU
* Xen
* Microsoft Hyper-V
* VMware ESXi

There are smaller projects out there that are not a subset of the above platforms, but I want something well supported. We can cross of ESXi and Hyper-V due to licensing. That leaves us with KVM or Xen. KVM is the most popular platform out there, powering GCP, AWS, and every other major platform out there. Xen is another option, but I've heard about negative changes to the licensing model and community support is a big thing. So, KVM it is (for now).

There are a number of frontends to KVM like QEMU out there. But, for the sake of this project I wanted to go with something bleeding edge- which is where Firecracker from AWS comes in.

It took me a minute to realize this, but Firecracker is a "one instance per VM." One firecracker process runs one VM. That makes sense, but not something that was easy to figure out from a quick read of the documentation. 

Firecracker is designed to be almost a VM, but almost a container. It visualizes the bare minimum hardware required for a VM, and starts the linux kernel directly. 

While Firecracker is cool (and fast), I wanted something with a bit more flexibility to run workloads. Mostly- run Windows. And that is not something Firecracker can do. The scope of this project does not include figuring out how to boot the Windows Kernel directly. 

Thankfully, there is an opensource project called Cloud Hypervisor which is a spiritual fork Firecracker, but with the goal of fleshing out a more feature complete hypervisor. 

But I'm getting ahead of myself a bit. I first tried this using QEMU, as it's well documented and great for figuring out the concepts. 

## Networking

## Orchestration 

## Scheduling

Scheduling is another area that I need to figure out.

The main products I looked at were Nomad, Kubernetes, and Mesosphere- why these and not more traditional HPC Schedulers like SLURM? The main reason is that I'm not familiar with them. 

### Nomad

The biggest reason I skipped Nomad is that it does not allocate CPU cores- only MHz. While this can work for containerized workloads, it does not translate well to Virtual Machines. If I want to allocate 2 cores to a VM, there is no way to effectively do that with Nomad. It would be possible if you had a homoogeneous cluster, but that is not something that is realistic to have nor something I have access to in my Home Lab. 

There is some work to add in full core scheduling, but at the time of this writing, is not released. 

### Kubernetes

Kubernetes solves the issues Nomad has when it comes to resource allocation, but has a new set of issues. 

The big one is complexity of the system in part due to the amount of batteries that are included. 

### Mesosphere

I avoided Mesos for two main reasons. The first is Indeed's experiences with Mesos & Marathon (where I worked when I started this project). It worked well for our needs, but we started to see issues with the various components as we tried to scale up. Development of Marathon has also slowed down, and there are some questions about the long term viability of the total project. 

The second is the lack of developer documentation out there. For this project, I would be interfacing with internal APIs to add in functionality, and the docs leave room for improvement. The code quality is good, so learning it is not impossible- but it would be an up hill battle to get it working

But. But but but. The way I'm designing this product allows for any of the above schedulers to be used. I'll go into this below.

# First Steps

## Go go QEMU

## Cloud-Hypervisor

# Making Design Choices

When developing a project, I believe the best way to approach something large like this is rapid iterations. 

There is a balance between making short term decisions to keep you moving forward, but also keeping in mind where you want to be later on in the project.

How does that apply here? Each component is it's own independent module, with a supporting library that is reusable. 

My QEMU Prototype is a simple project where everything is hard-coded. The goal is to figure out the basic concepts that I will need to know later and to have a starting point to see how to put the pieces together. 

Next up is transforming that working QMEU example into a working cloud-hypervisor example. The core logic of creating networks, disks, and booting the hypervisor is done already. Now, we just need to swap out QEMU for cloud-hypervisor. 

The process of getting that VM booted in QEMU, and then under cloud-hypervisor informs of the distinct steps that are needed in order to get a guest running. They work out to be something like the following.

* Disk Management
  * Creation of Metadata Disk (cloud-init metadata)
  * Ensuring disk images exist on disk- metadata, os, EFI firmware
* Network Management
  * Creation of macvtap device
  * Connection of macvtap to parent interface
  * Opening of File Descriptor and managing lifecycle 
* Hypervisor Management
  * Spawning of the Hypervisor
  * Configuring Hypervisor
  * Managing Guest Power State

Layering on the idea that disks/volumes will be stored remotely, and networks will be isolated means we need to make sure we can accommodate pre & post flight actions to setup and teardown those systems. 

Using that list above, it becomes clear that we have three distinct functions of this system. We have a Disk Management module, Network Management, and then Hypervisor management. 

By making sure there is this logical isolation between the various modules means down the road it will be easy to add additional functionality to each component that is outside the scope of this initial project. For my uses, I only need to support flat networking and remote volumes over NFS.