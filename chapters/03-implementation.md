# Implementation

For the solution to be usable in a real-life scenario, it had to be scalable, meaning that managing the bandwidth of 10 pods should take roughly the same amount of effort as managing thousands.
So when implementing the program, a great emphasis was placed on ease-of-usage and automation.
The user only needs to specify the bandwidth once, in it's yaml file that will be used to create it.
Afterwards, the program will make sure that each new pod created with that pod description will have it's bandwidth limited to the specified value.

The bandwidth can be specified with a special label in the pod's yaml file.
When the file is used to create a new pod, it is sent to the master node, which notifies an availabe worker node.
The worker node creates the pod, at which point a new cgroup, belonging to the newly created pod, appears.
At the same time the cgroup is created, a *pod creation* event is fired.
This event will cause the ratelimiter, running on the master node, to generate the eBPF program with the bandwidth limitation specified in the yaml file.
This eBPF program is sent to the worker node which started the pod.
A high-level overview can be seen on the following image.

![Creating a new pod inside the cluster.](../images/kubernetes.png)

The main idea that served as the primary goal for the projects was the attachment of the BPF program to the pod's cgroup.
After the eBPF program is sent to the appropriate worker node, the ratelimiter creates this attachment.

![Cgroup hierarchy in relation to pods](../images/node.png)

At this point, the program is loaded into the kernel, and the ratelimiting is already active.

## Token Bucket Algorithm

The ratelimiting is implemented with the *token bucket* algorithm, which is an algorithm used specifically for limiting bandwidth and burstiness on packet switched networks.
It works by the analogy of a *bucket*, that gets constantly filled with *tokens*.
Tokens are removed from the bucket at a constant rate (bandwidth).
If the bucket is full, the packets are dropped.

```c
static __s64 tokens = 1250000;
static __u64 time = 0;
static const __u32 bps = 125000; //speed in bytes/sec
static const __u32 extra_tokens = bps >> 9; //few extra tokens for smoother operation

int pkt_tbf(struct __sk_buff *skb)
{
	__u64 now = bpf_ktime_get_ns();
	__u64 new_tokens = (bps * (now - time)) / 1000000000;
	tokens += (new_tokens + extra_tokens);
	tokens -= skb->len;
	time = now;
	if(tokens > bps)
		tokens = bps;
	if(tokens < 0)
	{
		tokens = 0;
		return DROP_PKT;
	}

	return ALLOW_PKT;
}
```

In the case of TCP traffic, the dropping of packages results in those packages having to be resent by the sender.
TCP's inbuilt mechanism for dealing with packet loss will cause it to adjust it's rate or transmission, which will eventually be identical to the bandwidth limit created by the token bucket algorithm.

In contrast to TCP, dropping these packets in UDP traffic results in information being lost, as in UDP, no validation of the packet being received is done, so it won't get resent.
