# Introduction

Today, as cloud computing is becoming more and more ubiquitous, there is an increasing demand for finer control of resources in our data centers.
One of these resources if network bandwidth, TODO

## Motivation

The primare goal of this thesis is to provide an easy-to-use solution for limiting the bandwidth in our Kubernetes cluster on a per-pod basis.
To achieve this, it uses a technology called BPF (Berkeley Packet Filter), or as nowadays called, eBPF.
eBPF gives you the ability to run mini programs on a wide variety of kernal and application events. 
It makes the kernel programmable for people without background in kernel development.

## Thesis Structure

The thesis consists of multiple chapters, each TODO

# Development Environment

## Qemu

When I started the project, I used Qemu for testing newer kernel functions.
As most of the features that the project requires are still not widely adopted, I had to compile the kernel with the required features explicitly turned on.
Qemu made it convinient to test kernel features without affecting my own host operating system.
TODO ubuntu cloud image?

## Vagrant

At a later phase, when the project's direction was sufficiently clear, I switched to Vagrant.
Vagrant automates most of the steps required to bring up a suitable environment (like automatically syncing the project's directory to the VM).

## Docker

I chose Docker as the container runtime, mainly because its wide adoptation in the industry, and the number of available resources for it.
As of November 2020, the official release of the docker daemon still not supports the second version of *cgroups*, which my solution relies on, so I had to use the development version.

## Kubernetes

As it turns out, installing and configuring a Kubernetes cluster is not a trivials tasks.
That's the reason why for example Minikube or Microk8s exists.
These tools make it easy to have a fully functional cluster with minimal or even zero configuration.
Due to the nature of my project, I couldn't use these off-the-shelf solutions, so I had to create a working cluster myself.

To create the cluster, first the required binaries have to be installed.
There are two ways to create a cluster: you can do it manually, or you can use `kubeadm`.
Because time was pressing, and because I wouldn't have gained any additional benefits from creating the cluster by hand, I chose the second option.
On Arch linux[^fn13], the following packages had to be installed:

- `kubectl-bin`
- `kubelet-bin`
- `kubeadm-bin`
- `cni-plugins`
- `conntrack-tools`

The `kubelet-bin` and `kubeadm-bin` packages have to be installed on each node in the cluster.
These two programs will be used to create and maintain the cluster.
The `kubectl-bin` package contains the `kubectl` tool that can be used to communicate with the Kubernetes API.
`cni-plugins` and `conntrack-tools` are needed for networking inside the cluster.

When all the required packages are installed, there are two more requiremenets that needs to be fulfilled before using `kubeadm` to do most of the work.
First, swap has to be turned off.
This is becauser the Kubernetes scheduler, that's tresponsible for the availability of the pods, should know about the swap configurations for it to be ale to guarantee a pod's availabilty.
This would needlessly complicate things, so the community decided that swap support will be deferred for a later date, when the advantages of this feature would be big enough.[^fn11]
Swap can be turned off with:

`# swapoff /dev/<swap_parition>`

After swap is turned off, the kubelet service has to be started.
With `systemd`, this can be done with:

`# systemctl start kubelet.service`

To have the service start automatically after reboots, enable the service:

`# systemctl enable kubelet.service`

Now `kubeadm` should be able to initialize the cluster:

`# kubeadm init --apiserver-advertise-address=0.0.0.0 --ignore-preflight-errors=SystemVerification`

Here, the `--apiserver-advertise-address` option tells the kubeadm where should the cluster's API listen for incoming messages.
The `--ignore-preflight-errors=SystemVerification` line tells the kubeadm to ignore some errors, which was # TODO

After kubeadm finished with initializing the cluster, it will print some commands for configuring kubectl to use the cluster's API.

At this point, the cluster should already be running, but as the last step, it's recommended to install a CNI plugin.
CNI (Container Networking Interface) is the specification that containers use to communicate with eachother. Kubernetes uses this specification to make communicatoin between the pods inside the cluster possible.[^fn12]
By default, Kubernetes comes with its own CNI plugin, called kubenet.
The only problem is that this plugin has very limited capabilities, so it's always a good idea to install a more advanced CNI plugin.
Because CNI is only a specification, there are multiple ways to implement it.
This is why many CNI plugins exist, each implementing the specification in a different way.

The CNI plugin has to implement the following rules:

- All containers can communicate with all other containers without NAT
- All nodes can communicate with all containers (and vice-versa) without NAT
- The IP that a container sees itself as is the same IP that others see it as

The plugin I chose was Weave Net, which can be installed after kubectl has been configured with the following command:

`$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

If everything went according to the plan, the cluster is running and fully functional now.

# Implementation

The main idea that served as the primary goal for the projects was the attachment of the BPF program to a cgroup.
In the final version, this is the pod's cgroup that want to impose the bandwidth limit on.
However, in earlier versions I simply attached the BPF program to the root cgroup, so I could easily test and see the effect without the need to figure out the exact cgroup to use and measure the bandwidth from the cgroup's context.

## First Version

In the first version of the project, I deliberately left out all the Kubernetes related TODO stuff, and only concentrated on sending and attaching a BPF program to the VM.
This meant that I was able to test and provide a proof that the concept was indeed TODO working, without dealing with all the complexities of a Kubernetes cluster.
In this initial Proof of Concept, the project consisted of two separate programs, that will be described in detail below.

### Client

The client was a command line tool, with the main purpose of attaching and detaching a BPF program to the VM.
In the early stages, the BPF program it sent to the daemon was only marking the individual packets.
That was later replaced with the *shaper* used in the current version.
With the tool, the user could *attach* and *detach* a BPF program and *get the status* of the VM (does it have a program attached).
While this initial version was not the direction the project went towards later (the command-line tool was replaced with a background process that requries no user interaction), it still helped in clarifying the user requirements and providing a backbone that the next version could be built upon.

### Daemon

The daemon runs on the VM, waiting for messages from the client.
Based on the message it receives from the client, it attaches or detaches the BPF program, then sends a response to the client.
Compared to the client, the daemon remained mostly intact in the final version.

## Second Version

In the second version, the program had to be integrated into the context of a Kubernetes cluster.
The first task was to get a working Kubernetes cluster going.
My first idea was to use Minikube, which pre-configured Kubernetes cluster pre-packaged as a container or a VM.
Unfortunately this proved to be incompatible with my solution, so I had to install and configure the cluster on my machine as described above.
After the cluster was set up and running, the previous solution could be repurposed to work with Kubernetes.
Mostly the client was affected, as it now was communicating with the Kubernetes API, and not with the user.
This required it to continually run in the background, waiting for events coming from the cluster.

# User Documentation

## Dependencies

In this section, I go through the dependencies that are required by the project.
I'm assuming that a Kubernetes cluster is already set up, so I only describe the changes necessary for my project to work, although the Kubernetes section inside the development environment chapter describes how to install and configure a Kubernetes cluster if needed.

- python kubernetes library TODO add requirements

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
The sources of bpftool can be found in the kernel repository.[^fn8]
It can be installed with `make install`.

### Docker

After cgroup v2 is mounted, the Docker daemon will most likely fail to start.
This is due to the fact that most container runtimes, including docker, still don't support the new cgroup hierarchy.[^fn4]
Fortunately, the project's master branch[^fn5] already contains the necessary changes for cgroup v2 support.
Compiling it and using the resulting `dockerd-dev` binary solves the problem.

To compile the newest version, clone the respository at https://github.com/moby/moby, then use the Makefile that came with it to compile and install it:

`# make binary`

### LLVM, Clang

For compiling the the BPF program, my solution uses the *LLVM*[^fn7] compiler collection, although GCC should have BPF support too[^fn6].
For a frontend to LLVM, it uses *clang*.

### Kubernetes Python Client

The Kubernetes Python Client[^fn9] is the official client library for Kubernetes.
The `requirements.txt` file inside the project's folder contains the exact version I used, so it can be installed with:

`$ pip install -r requirements.txt`

Alternatively, the newest version can be installed with:

`$ pip install kubernetes`

## Usage

To use the program, a Kubernetes cluster has to be running already.
The `ratelimit_master.py` file contains the script that should be started on the master node, while `ratelimit_slave.py` contains the code that runs on the worker nodes.
While the `ratelimit_master.py` script can be executed by any user that's able to communicate with the Kubernetes API, `ratelimit_slave.py` can only be executed by root, because attaching BPF programs requires higher privileges.

The program requires no further actions from the user, the program will do the attaching and detaching of the BPF programs automatically when it detects the starting and stopping of pods in the cluster.

The rate limit that the program will assign to the pod, can be specified with a **label** in the pod's YAML file.
The label that the program will be looking for is `rate`, and it's value must contain the desired bandwidth in the following format: `"<limit in Megabytes>M"`.
For reference, here is an example YAML file, that describes a pod named *test*, with the bandwidth limit of 1 MB/s:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
  labels: 
    component: web
    rate: "1M"
spec:
  containers:
    - name: test
      image: dashsaurabh/progressive-coder
      image: quay.io/openshiftlabs/simpleservice:0.5.0
      ports: 
      - containerPort: 9875
```


# Developer Documentation

The newest version can be found at https://github.com/EarlPitts/bpf-ratelimit.
The projects contains the following modules and files:

## `ratelimit_master.py`

This module contains the logic that runs on the cluster's master node.
When its started, it first loads the cluster's configuration, then it begins watching for events in the cluster.
If it detects the starting or stopping of a pod, the corresponding function is triggered.
If the event it detected was the deletion of pod, it sends a message to the right node, that the pinned BPF program should be removed.
In the case of a pod creation event, first the BPF program with the value for rate limiting, which is given in the pod's configuration file is created, then it's sent to the node, where the pod was created, together with information that tells the node how to attach the program.

## `ratelimit_slave.py`

The `ratelimit_slave.py` module runs on each worker node.
When it's started, it creates a server socket, then blocks, waiting for incoming connections.
Based on the message it gets from the establised connection with the master node, either the `__attach()` or `__detach()` function is executed.
When the master node want's to attach a new BPF program, it sends the pod's unique ID, the size of the BPF program and the BPF program itself to the slave.

The pod's ID tells the slave where should the program it received be attached.
Kubernetes gives every pod it manages a unique ID.
This ID can be used to find the pod's *cgroup*.
When Kubernetes is running, a cgroup is created in the `/sys/fs/cgroup` pseudo-filesystem named `kubepods.slice` (assuming cgroup v2 is used).
In this cgroup, there are additional cgroups for each *pod QoS class*, namely:

- Guaranteed
- Burstable
- BestEffort

Finally, inside these cgroups are the cgroups belonging to the actual pods.
The full path to a pod's cgroup could look something like this:

`/sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod5a80687a9d89b9e0b9361cf37a77ade7.slice`

This could be elaborated further still, as the individual pods' cgroups contain the cgroups of the containers that are inside the pod, but that's outside the scope of this project.

The size of the BPF program is sent next, because without it, the slave wouldn't know when should it stop expecting more packages containing the BPF program.
After the program is sent to the slave, it places it in the `/sys/fs/bpf` directory, then tries to attach it to the cgroup of the pod.
To avoid name collisions if multiple BPF programs are sent to the same machine, the programs are placed inside their own folder, for which the following naming convention is used:

`/sys/fs/bpf/<unique-id-of-pod>`

At almost every stage of the communication between the master and the slave, the slave sends back a response, telling the master if it can proceed with the next step, or some error occurred.

Compared to attachment, the detachment of the program is much more straightforward, requiring only a few steps.
As the cgroup of the pod is taken care of by Kubernetes, the only remainig artifact is the BPF program inside `/sys/fs/bpf`.
First, similar to the attachment, the unique ID of the pod is sent by the master.
With this information, the slave removes the remaining BPF program.

## `bpf_generator.py`

This module is responsible for generating the BPF program that will be attached to the pod's cgroup.
It contains the source code of the BPF program that will be generated, with the network rate part left out, so it can be dynamically filled in by the script.

## `bpf_helpers.h`

The `bpf_helpers.h` header file contains some helper functions that are needed for compiling the BPF program.
The `bpf_generator.py` module links to it while generating the program.

## Architectural Overview

# Testing

The make sure that the project was proceeding in the right direction, the program has to be tested at each stage of the development process.
As the project was moving forward, testing the program to see if it produces the desired functionality was becoming more and more elaborate.

## Marker

The first goal of the project was to be able to compile and load the BPF program into the kernel in a suffieciently consistent manner, as it was the foundation of everything that followed.
This was done using the host operating system, using a BPF program that marked the outgoing packets.
These packets could be observed with a tool like `tcpdump`, or rules could be added with `tc` to only allow the marked packets to leave the machine.

Next, the same was done using a virtual machine.
With `tc`, rules were declared on the virtual machine that only allowed the marked packets to get through.

## Shaper

At the next stage of the project, the marker got replaced with the *shaper* program, that would remain in the final solution.
This meant that not the packet itself, but the number of packets leaving the machine were affected, so the prvious testing method couldn't be used anymore.
To test if the shaper was working as expected, various network monitoring tools (like `bmon`[^fn14]) and a tool called speedtest-cli[^fn15] were used.
These tools were able to precisely measure the incoming and outgoing rate, which could be used to tell if the shaper was working correctly.

## Containers

Under the hood, if you remove the multiple layers of abstractions, you will find that Kubernetes uses the same old container concept that you are probably familiar with.
So the next logical step was to test the shaper on a container.
To achieve this, the shaper was attached to the cgroup of the container.
Then the same tools were used to test the bandwidth, only this time it had to be done from the context of the container.
Fortunately, the docker frontend provides multiple commands that make this task almost trivial.
The `docker exec` command can be used to execute arbitrary command inside the container.
With the `-i` and `-t` flags it can even make this "interactive" (meaning that STDIN will remain open even it the container is not attached) and a pseudo-tty can be allocated respectively.
The command expects a container as argument, and a command that we want to execute inside the container.
This way, we can easily get a shell inside the container with the following command:

`docker exec -it <container_id> sh`

> *If the container has a more user-friendly shell like bash, that could be used instead.*

Now that we are inside the container, the aforementioned tools can be easily installed with the help of the available package manager.
With the tools present, the same measurements can be taken as before, confirming that the shaper functions correctly with the container.

# Summary

# Results

Cilium, a company providing eBPF based solutions for networking, announced its newest release, which introduced a new feature that does almost exactly the same thing as my project.[^fn10]

The project was a great opportunity to learn about networking concepts more in-depth. TODO

# Further Plans

TODO BCC, XDP
TODO native python, not bpftool
TODO test for multiple nodes
TODO Store state

# Sources

[^fn1]:https://kernelnewbies.org/Linux_3.16#Unified_Control_Group_hierarchy
[^fn2]:https://kernelnewbies.org/Linux_3.16#Unified_Control_Group_hierarchy
[^fn3]:https://wiki.archlinux.org/index.php/Kernel_parameters
[^fn4]:https://medium.com/nttlabs/cgroup-v2-596d035be4d7
[^fn5]:https://github.com/moby/moby
[^fn6]:https://llvm.org/
[^fn7]:https://lwn.net/Articles/800606/
[^fn8]:https://twitter.com/qeole/status/1113121896617447424
[^fn9]:https://github.com/kubernetes-client/python
[^fn10]:https://cilium.io/blog/2020/11/10/cilium-19#bwmanager
[^fn11]:https://medium.com/tailwinds-navigator/kubernetes-tip-why-disable-swap-on-linux-3505f0250263
[^fn12]:https://medium.com/@ahmetensar/kubernetes-network-plugins-abfd7a1d7cac
[^fn13]:https://wiki.archlinux.org/index.php/Kubernetes
[^fn14]:https://github.com/tgraf/bmon
[^fn15]:https://www.speedtest.net/apps/cli
