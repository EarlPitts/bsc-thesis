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

On both the master node and the worker nodes, a BPF compatible kernel has to be installed.

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
