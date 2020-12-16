# Introduction

For cloud computing to become how we know it today, it had to go through a series of steps, with each TODO
When mainframes first started to appear, their astronomical price made it practically impossible for companies to give each of their employees a device of their own.
However, having only a single person operate the mainfram would lose precious compute time.
To solve the problem of allocating the mainframe's resources more efficiently, time-sharing was invented.
This allowed employees to use the same hardware in parallel, reducing idle time.

The next big leap in the process of maximizing resource utilization was virtaulization.
One of the goals of virtualization was to distribute these finite resources in the most efficient way possible.
Virtualisation works by simulating virtual hardware, called a *Virtual Machine* (VM), which then can be used to run a guest operating system on.
The parameters of this virtual machine, like CPU, memory, disk space, can be freely configured (limited only by the resources available to the host), and even the network interface controller (NIC) or various networking devices can be simulated or shared with the VM.
The problem with virtualization is the need to simulate the whole operating system in each virtual machine.

The idea of the VM was taken further with the introduction of operating-system level virtualization.
This approach provided a solution for the problem of needing to unnecessarily simulate the whole operating system.
In this model, there are multiple user-space "instances", isolated from eachother, sharing the same kernel.
These instances are what we call containers.
On Unix-like systems, this abstraction of the filesystem and the resources is done by a more advanced implementation of the *chroot* mechanism, called *namespaces* and a resource-management function, provided by the kernel, called *cgroups*.

The advent of containers have paved the way for the *microservice* model.
In this model, each part of the software acts as a self-contained service, that can be easily swapped with another one if needed.
The benefits of this approach are numerous, including robustness (if a container malfunctions for some reason, an other can be placed in its place), scalability (more copies of the service can be started or exisiting copies destroyed easily at will), modularity (these services are losely coupled, they can be easily replaced with newer version of the same service, or even a completely different service), etc...

Managing these containers is tiresome, and quickly becomes an almost impossible task as the application grows and becames more complex.
A solution was needed that would automize the management of these services, making it easier for developers to concentrate on the application itself.
The solution came in the form of the *container orchestration* tools.

The most widely used container orchestration tool is arguably Kubernetes, which was origianlly designed by Google.
Kubernetes has the ability to automate most tasks related to the management of containers, like creating and destorying containers based on need.

For these containers to be able to communicate with eachother inside the cluster, a complete virtual network has to be created.
This virtual network will be the main focus of this thesis.

## Motivation

The primary goal of this thesis is to provide an easy-to-use solution for limiting the bandwidth between pods in our Kubernetes cluster in a controlled way.
To achieve this, it uses a technology called BPF (Berkeley Packet Filter), or as nowadays called, eBPF.
eBPF gives you the ability to run mini programs on a wide variety of kernal and application events. 
It makes the kernel programmable for people without background in kernel development.

For networking capabilities, Kubernetes relies on third-party plugins called CNI (Container Network Interface).
These plugins make it possible for containers to connect to other containers, the host or outside the network.

Cilium, flannel, weave

## Thesis Structure

The thesis consists of multiple chapters.
Chapter 2 contains some remarks about the development environment and the technologies I used in the project.
Chapter 3 elaborates on the approach I used when implementing the software.
TODO
Chapter 4 contains the user documentation.
It describes the required dependencies for the project, gives some instructions for setting up and configuring necessary components of the system and explains how to use the software.
Chapter 5 is the developer documentation.
This chapter iw written for developers, and its main goal is to help them understand the inner workings of the software, making troubleshooting or even extending the project with new functionalities easier.
Chapter 6 describes the various tecniques used for testing the software at each stage of the development process.
Chapter 7 gives the reader a summary about the outcome of the project.
