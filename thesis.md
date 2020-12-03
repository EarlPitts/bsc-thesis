# Introduction

Today, as cloud computing is becoming more and more ubiquitus, there is an increasing demand for finer control of resources in our data centers.
Network bandwidth is a valuable resource, TODO
The primare goal of this thesis is to provide an easy-to-use solution for limiting the bandwidth in our Kubernetes cluster on a per-pod basis.

## Motivation

Provides a solution to limit the incoming bandwidth on a per-pod basis.
To achieve this, it uses a technology called BPF (Berkeley Packet Filter), or as nowadays called, eBPF.
eBPF gives you the ability to run mini programs on a wide variety of kernal and application events. 
It makes the kernel programmable for people without background in kernel development.

## Thesis Structure

The thesis consists of multiple chapters, each 

# Development Environment

## Qemu

When I started the project, I used Qemu for testing newer kernel functions.
As most of the features that the project requires are still not widely adopted or even in development status, I had to compile 
Qemu made it convinient to test kernel features without affecting my own host operating system.
TODO ubuntu cloud image?

## Vagrant

At a later phase, when the project's direction was TODO eleg clear, I switched to Vagrant.
Vagrant automates most of the steps required to bring up a suitable environment (like automatically syncing the project's directory to the VM).

## Docker

I chose Docker as the container runtime, mainly because its wide adoptation in the industry, and the number of available resources for it.
As of November 2020, the TODO official docker daemon still not supports the second version of *cgroups*, which my solution relies on, so I had to use the development version.

## Kubernetes



# Implementation

TODO bpf attached to cgroup

## First Version

In the first version of the projects, I deliberately left out all the Kubernetes related TODO stuff, and only concentrated on sending and attaching a BPF program to the VM.
This meant that I was able to test and provide a proof that the concept was indeed TODO working, without dealing with all the complexities of a Kubernetes cluster.
In this initial Proof of Concept, the project consisted of two separate programs, that will be described in detail below.

### Client

The client was a command line tool, with the main purpose of attaching and detaching a BPF program to the VM.

### Daemon

The daemon runs on the VM, waiting for messages from the client.
Based on the message it receives from the client, it attaches or detaches the BPF program, then sends a response to the client.

## Second Version

In the second version, the program had to be integrated into the context of a Kubernetes cluster.
The first task was to get a working Kubernetes cluster going.
My first idea was to use Minikube, which pre-configured Kubernetes cluster pre-packaged as a container or a VM.
Unfortunately this proved to be incompatible with my solution, so I had to install and configure the cluster on my machine.



- cgroups
- kubernetes

# User Documentation

## Dependencies

### Kernel

On both the master node and the worker nodes, a BPF compatible kernel has to be installed.
The distribution I've used was *Arch*, which ships with a kernel that has all the necessary kernel features enabled by default.

For distributions which don't have the necessary features enabled by default, the following configuration can be used to compile the kernel:

```
CONFIG_CGROUPS=y
CONFIG_BLK_CGROUP=y
CONFIG_DEBUG_BLK_CGROUP=y
CONFIG_CGROUP_SCHED=y
CONFIG_CGROUP_PIDS=y
CONFIG_CGROUP_RDMA=y
CONFIG_CGROUP_FREEZER=y
CONFIG_CGROUP_HUGETLB=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_CPUACCT=y
CONFIG_CGROUP_PERF=y
CONFIG_CGROUP_BPF=y
CONFIG_CGROUP_DEBUG=y
CONFIG_SOCK_CGROUP_DATA=y
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_BLK_CGROUP_IOLATENCY=y
CONFIG_IPV6_SEG6_BPF=y
CONFIG_NETFILTER_XT_MATCH_BPF=y
CONFIG_NETFILTER_XT_MATCH_CGROUP=y
CONFIG_BPFILTER=y
CONFIG_BPFILTER_UMH=y
CONFIG_NET_CLS_CGROUP=y
CONFIG_NET_CLS_BPF=y
CONFIG_NET_ACT_BPF=y
CONFIG_CGROUP_NET_PRIO=y
CONFIG_CGROUP_NET_CLASSID=y
CONFIG_BPF_JIT=y
CONFIG_BPF_STREAM_PARSER=y
CONFIG_LWTUNNEL_BPF=y
CONFIG_HAVE_EBPF_JIT=y
CONFIG_BPF_EVENTS=y
CONFIG_BPF_KPROBE_OVERRIDE=y
CONFIG_TEST_BPF=m
```

### Cgroup V2

Although cgroup version 2 was in the kernel as early as 2014 [^fn1], most distributions still hasn't adapted it, and use the first version by default.
You have to explicitly tell the kernel that you want the newer version.
To do this, a *kernel parameter* has to be passed at boot time:

`systemd.unified_cgroup_hierarchy=1`[^fn2]

There are multiple ways to pass a parameter to the kernel, but for our purpose it makes the most sense to do it through the boot loader.
For most users, this will be GRUB, so I'll only show how to do it with GRUB, but information regarding other boot loaders can be found on the Arch wiki[^fn3].

When GRUB starts, and asks you to select the OS you want to boot into, you can press `e` to access the configuration.
The kernel parameters have to be placed at the and of the `linux` line:

`linux   /boot/vmlinuz-linux root=UUID=bf3f606b-e532-4b8a-a552-f90bafd14776 rw loglevel=3 quiet systemd.unified_cgroup_hierarchy=1`

Pressing `F10` will boot with the modified configuration.

Configuring the kernel in this way will only work until the system is rebooted, at which point the default parameters will be used again.
To make the change persistent, the GRUB configuration file has to be updated, which can be found at `/boot/grub/grub.cfg`.

Although it's possible to manually edit this file, this is usually not recommended, because a small typo could make the system unable to start.
The proper way to update GRUB configurations is to first modify the `/etc/default/grub` file, which is specifically used for user overrides, then use GRUB's own tool to generate the configuration file.

The same `linux` line has to be placed inside the `/etc/default/grub` file that was used previously, then the `grub-mkconfig -o /boot/grub/grub.cfg` command can be used to generate the configuration and override the previous one.

### Bpftool

My current solution relies on `bpftool`, which is a tool used for managing BPF programs.
Without it, the `bpf()` system call would have to be called manually.

### Docker

After cgroup v2 is mounted, the Docker daemon will most likely fail to start.
This is due to the fact that most container runtimes, including docker, still don't support the new cgroup hierarchy.[^fn4]
Fortunately, the developer 

- bpftool
- docker-dev
- cgroup v2
- kernel

## Usage

# Developer Documentation

## Architectural Overview

# Testing

# Further Plans

TODO native python, not bpftool
TODO test for multiple nodes

# Summary

[^fn1]:https://kernelnewbies.org/Linux_3.16#Unified_Control_Group_hierarchy
[^fn2]:https://kernelnewbies.org/Linux_3.16#Unified_Control_Group_hierarchy
[^fn3]:https://wiki.archlinux.org/index.php/Kernel_parameters
[^fn4]:https://medium.com/nttlabs/cgroup-v2-596d035be4d7
