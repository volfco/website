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

The big one is complexity of the system, and building on top of it. If you use kubernetes, you really need to go all in to properly take advantage of it.

### Mesosphere

I avoided Mesos for two main reasons. The first is Indeed's experiences with Mesos & Marathon (where I worked when I started this project). It worked well for our needs, but we started to see issues with the various components as we tried to scale up. Development of Marathon has also slowed down, and there are some questions about the long term viability of the total project.

The second is the lack of developer documentation out there. For this project, I would be interfacing with internal APIs to add in functionality, and the docs leave room for improvement. The code quality is good, so learning it is not impossible- but it would be an up hill battle to get it working

But. But but but. The way I'm designing this product allows for any of the above schedulers to be used. I'll go into this below.

### Others

I want to look at using SLURM down the road, 

[https://en.wikipedia.org/wiki/Comparison_of_cluster_software](https://en.wikipedia.org/wiki/Comparison_of_cluster_software "https://en.wikipedia.org/wiki/Comparison_of_cluster_software")

# First Steps

## Go go QEMU

The basic process for spawning a guest with QEMU is:

1. Create your base qcow2 image from either an existing cloud-image, or some other way.
2. Resize it with `qemu-img` to the desired size
3. Create a FAT16 or ISO disk with cloud-init metadata
4. Configure Networking (or let QEMU do it for you)
5. Launch QEMU with the proper configuration

### Base OS

For one, it's easy. We'll use an ubuntu cloud-image from Ubuntu directly. Prebuilt and perfect for our needs. 

### Bask Disk

Step two is also easy. QEMU provides a qemu-img utility that can do all the disk operations we need it to.

    qemu-img resize <image path> 20G

### Metadata

Three gets a bit more complex. We need to generate cloud-init configuration, and then make a Fat16 or ISO disk image that the VM can read on boot. I'm not going to go into what goes into a cloud-init config disk here, as there is information in later articles and the gist linked. 

    genisoimage -output cloud-init.iso -volid cidata -joliet -rock user-data meta-data

This spits out a cloud-init.iso file that we will need to attach to QEMU _before_ the base OS disk.

_I used_ [_https://gist.github.com/ebal/03b9f00e6412971d3fd5b0b685b1c95d_](https://gist.github.com/ebal/03b9f00e6412971d3fd5b0b685b1c95d "https://gist.github.com/ebal/03b9f00e6412971d3fd5b0b685b1c95d")  _as a reference for the creation of the ISO._ 

### Networking

### Spawning

Once we have the other steps done, we can boot up the guest and hope it works. 

```rust
pub fn run_qemu_machine(vm: structs::VM) -> bool {
    let mut qemu_args: Vec<String> = Vec::new();

    // Base options
    qemu_args.push("-enable-kvm".to_string());
    qemu_args.push("-nographic".to_string());

    // CPU Type
    qemu_args.push("-cpu".to_string());
    qemu_args.push(vm.cpu_type.clone());

    // CPU Cores
    qemu_args.push("-smp".to_string());
    qemu_args.push(format!("cpus={},sockets=1", vm.cpu_cores));

    // Memory
    qemu_args.push("-m".to_string());
    qemu_args.push(format!("size={}", &vm.ram_total));

    // Console
    qemu_args.push("-monitor".to_string());
    qemu_args.push(format!("unix:/gn1/run/{}.sock,server,nowait", &vm.id));

    // Serial Log
    qemu_args.push("-serial".to_string());
    qemu_args.push(format!("file:/gn1/run/{}.log", &vm.id));

    // use the first volume's ID as the metadata location. we're making an assumption, but considering
    // we're explicitly putting metadata there... it's not a problem
    // TODO we should gracefully fail here anyways
    qemu_args.push("-drive".to_string());
    qemu_args.push(format!("file={},media=cdrom", format!("/gn1/vmfs/{}/disk-{}/metadata.iso", &vm.volumes\[0\].vmfs, &vm.volumes\[0\].id)));

    // now we loop over each disk. We're assuming they're in the correct order
    for volume in &vm.volumes {
        qemu_args.push("-drive".to_string());
        qemu_args.push(format!("file={},media=disk,if={}", format!("/gn1/vmfs/{}/disk-{}/disk.qcow2", volume.vmfs, volume.id), volume.driver));
    }

    for network in &vm.networks {
        // bridge,br=br1,model=virtio
        qemu_args.push("-nic".to_string());
        qemu_args.push(format!("bridge,br=brvx{},mac={},model={}", network.vxlan, network.mac, network.driver));
    }

    let process = process::Command::new("/usr/bin/qemu-system-x86_64")
        .args(qemu_args).spawn();

    if process.is_err() {
        error!("\[start_qemu_machine:{}\] unable to spawn VM. failed with the following error: {}", line!(), process.err().unwrap().to_string());
        return false;
    }

    let mut proc = process.unwrap();
    let pid = proc.id();
    info!("successfully spawned VM with pid {}", &pid);
    
    let exit = proc.wait_with_output();
    if exit.is_err() {
        error!("\[run_gn1_qemu_machine:{}\] IO error while waiting for process to exist. {}", line!(), exit.err().unwrap().to_string());
    } else {
        debug!("\[run_gn1_qemu_machine:{}\] vm exited successfully", line!());
    }
    return true;
}
```

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