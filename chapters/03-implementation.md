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
