# Introduction

For cloud computing to become how we know it today, it had to go through a series of steps, each improving upon the previous solution.
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
They do this by creating an *overlay network* on top of the alrady existing one.
An overlay network is just a virtual network, providing the same functionalities as a phisycal network.
It functions by encapsulating the network packets in an additional layer.

In the case of smaller clusters, pretty much any CNI plugin will do the job just fine.
Howerever, it's a good idea to pick the CNI plugin carefully right at the beginning, because when the cluster grows to contain more then just a handful of services, the plugin you chose can have a great impact on scaling it further, and changing it without breaking the cluster is usually not a trivial task.

There are plenty of CNI plugins to choose from, so choosing the right one requires quite a bit of research.
To mention just a few: Cilium, Flannel and Weave Net are among the most popular choices.
Flannel is the developed by CoreOS, and arguably the most popular one.
It's easy to install, and works by creating a layer 3 overlay network, in which each pod has a subnet for allocating IPs for its containers internally.
Weave uses a *mesh overlay network*, which means that each node in the overlay network is linked to multiple other ones.
Each host has a routing component installed, and these communicate with eachother, exchangeing information about the topology, so they all stay up to date.
Cilium uses eBPF to provide these functionalities and some security related ones.
This means that in a lot of cases, it's the fastest solution, because it can inspect and modify the packages inside the kernel, which in turn have huge implications in scaling the cluster.

The problem with current solutions is the lack of functionality regarding network resource management.
Network is a finite resource, so it would be great, if we could control the quality of the service provided to customers based on the amount they pay for it.
Currently, this is only possible by setting up rules manually, which is practically impossible when you have more than a couple of customers.
Furthermore, while providing this functionality inside the CNI plugin itself is possible, this would stop users of other CNI plugins, which doesn't have this function, to use it.
My solution provides this functionality regardless of the CNI plugin used, while still being easy to configure and use, using the *cgroup* and *eBPF* subsystems already provided by the kernel.

## Thesis Structure

The thesis consists of the following chapters:  
Chapter 2 contains the user documentation.
It describes the required dependencies for the project, gives some instructions for setting up and configuring necessary components of the system and explains how to use the software.
Chapter 3 elaborates on the approach I used when implementing the software.  
Chapter 4 contains some remarks about the development environment and the technologies I used in the project.  
Chapter 5 is the developer documentation.
This chapter iw written for developers, and its main goal is to help them understand the inner workings of the software, making troubleshooting or even extending the project with new functionalities easier.  
Chapter 6 describes the various tecniques used for testing the software at each stage of the development process.  
Chapter 7 gives the reader a summary about the outcome of the project.
