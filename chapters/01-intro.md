# Introduction

## Motivation

Today, as cloud computing is becoming more and more ubiquitous, there is an increasing demand for finer control of resources in our data centers.
One of these resources if network bandwidth, TODO
Whith the shift from yesterday's monoliths to the microservices framework
Microservices
Kubernetes



The primary goal of this thesis is to provide an easy-to-use solution for limiting the bandwidth in our Kubernetes cluster on a per-pod basis.
To achieve this, it uses a technology called BPF (Berkeley Packet Filter), or as nowadays called, eBPF.
eBPF gives you the ability to run mini programs on a wide variety of kernal and application events. 
It makes the kernel programmable for people without background in kernel development.

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
