# Summary

In this thesis, I tried to provide a way for network traffic shaping inside a Kubernetes cluster using the eBPF kernel feature.

## Results

Towards the end of the project, Cilium, a company providing eBPF based solutions for networking, announced a new release, which, as one of its new features, implemented the same functionality as the one my solution provides.[^fn10]
Although it was rather demotivating first, seeing that someone has done what I was working on, it nonetheless proved that the project was solving a real-world problem.

The project was a great opportunity to learn about networking concepts more in-depth.
It was also a good way to familiarize myself with current technologies in the cloud native sphere.

I think the test results show that the project was successful, and it delivers the promised functionalities.

## Further Possibilities

Although the project is in a usable state, there are plenty of room for improvement.
As closing thoughts, I would like to briefly mention some ways it could be made better.

### XDP

XDP is a specialized application, that processes packets at the lowest level of the software stack.
It's specifically designed for high performance.
The project currently isn't using XDP, so it's an obvious candidate for improvement.
With XDP, the eBPF program could be run basically on the NIC, making it much faster.

### BCC

There is a framework for creating BPF programs called BCC.
Although it would be another dependency, I think it's benefits outweight it's disadvantages.
