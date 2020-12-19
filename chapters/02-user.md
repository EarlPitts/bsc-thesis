# User Documentation

## Use Cases

The primary use case of the software is to limit the bandwidth between services inside the cluster.

## Dependencies

In this section, I go through the dependencies that are required by the project.
I'm assuming that a Kubernetes cluster is already set up, so I only describe the changes necessary for my project to work, although the Kubernetes section inside the development environment chapter describes how to install and configure a Kubernetes cluster if needed.

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
### Cgroup v2

Cgroups (or *control groups*) is feature, provided by the kernel, for limiting the amount of resources a process can use.
The resources that can be controlled by cgroups include CPU, RAM, block I/O, network I/O, etc...
You can use cgroups by manually modifying files inside the `/sys/fs/cgroup` pseudo-filesystem.
This folder contains the cgroup hierarchy which is just a series of folders nested inside eachother, with the root cgroup being at the top.
Cgroups in the lower end of the hierarchy share the resources of their parents, and can be further limited.
The init process, which has the PID 1, sits at the top of the hierarchy.
Processes can be added to cgroups by appending their PIDs to the `tasks` file inside the chosen cgroup.
Processes also inherit the cgroup membership from their parent process.
Because a deeper understanding of the inner workings of cgroups is not required for using the software, I won't go into greater details, but curious readers can easily find more information about cgroups on the internet.

My solution requires the second version of cgroups to work.
Compared to the first version, the main differences are the unified hierarchy and the thread granularity.
The unified hierarchy means that while in the first version, there were separate hierarchies for each resource types, v2 combines them into a single hierarchy.
In v1, there was thread granularity, meaning that individual threads were assigned to cgroups.
v2 changed this by only assigning processes to the cgroups.

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
