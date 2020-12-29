# Development Environment

For the initial experimenting phase and later testing if the function implemented was working properly, *Qemu* and *Vagrant* were used.
These tools helped by emulating the target machine, without the need to use real hardware.

## Qemu

As mentioned before, Qemu (Quick emulator) is a virtual machine emulator, similar to Virtualbox or VMWare.
It supports booting with a custom kernel directly from the host.
This eliminates the problem of copying the kernel inside the VM after every modification, or compiling the kernel inside the slower VM.
When I started the project, I used Qemu for testing newer kernel functions.
As most of the features that the project requires are still not widely adopted, I had to compile the kernel with the required features explicitly turned on.
Qemu made it convinient to test kernel features without affecting my own host operating system.

![Emulating the target device using Qemu](../images/qemu.png)

## Vagrant

At a later phase, when the project's direction was sufficiently clear, I switched to Vagrant.
Vagrant automates most of the steps required to bring up a suitable environment (like automatically syncing the project's directory to the VM).
This made it much easier and faster to experiment with new ideas and features.

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
