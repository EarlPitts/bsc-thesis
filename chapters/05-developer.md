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
