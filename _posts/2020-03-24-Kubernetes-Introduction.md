---
layout: post
categories: [Writeups]
comments: true
excerpt: Virtualization, Containerization and Serverless - a laymans introduction
title: How'd we get here - Serverless
tags: [docker, kubernetes, cloud native]
description: Virtualization, Containerization and Serverless - a laymans introduction
---


# Introduction

## Generally speaking

Since this is an online post, I'm assuming you're reading it on some kind of a computer, phone or tablet device. Let's start with what makes up the device. 


<div class="mermaid text-center">
graph TB
    subgraph Flow
    Application-->o1[Operating System]-->BIOS-->Hardware
    end
style Flow fill:#f1f1f1
style Flow stroke-width:0px
</div>


This is obviously over simplified however it'll help get the point across. Here's what happens when you run/ open a programme on your machine:

1. You log into the machine as a user (your iPhone user passcode, windows or Mac user). Every command you run after login is exceuted by your user. 
2. You double click an icon on your machine. The OS understands what to do next, such as opening a window like Chrome, Excel etc. At this point it needs to get information on what to show you and how such as the chrome browser layout or the excel formatting and the data inside it. 
3. To get this information it needs to access the harddisk but the language used by the Operating System isn't the same as the language used by the Harddisk!!
4. Hence the Operating System sends a message to the BIOS (Basic Input Output System) which then talks to the harddisk and gets the data and passes it back along. 


Today, we're used to running multiple processes on a single machine - such as a couple of chrome windows, Excel, Powerpoint etc. all at the same time. All these Applications share the same Operating System and BIOS code in order to run. 


## How did we get here?


When software developers started writing applications for consumers, such as a website, they had to run these codes on powerful machines (servers). They then wanted to run multiple applications on the same machine for optimal resource utilization, dependencies between applications etc. This however came with it's own set of complications:


1. **One OS per machine Limitation**: Applications would sometimes have specific Operating System requirements. For example some would be made to run on a Mac, Linux or Windows. 
   
2. **No seperation of concerns**: Individual applications would be owned by different teams and it would get extremely difficult to figure out if someone else messed with things that impacted another application. System restarts would also cause an impact on all the applications running on that server.
   
3. **Resource Utilization Inefficiency**: Applications could end up over using RAM or the storage of a machine impacting other applications running simultaneously.


### Multi-user OS


So server operating systems came up with the capability to seperate contexts for different users on the same machine. Think of it as how 2 users created on the same Windows or Mac can customize their desktop's look and feel, install different applications that don't interfere etc. Application developers could now login through their own specific users and control what ran using that user. This additionally brought the capability to have multiple users logged into the machine/ running applications in their own users context in parallel.

1. **One OS per machine Limitation** <i class="fa fa-times-circle text-danger" aria-hidden="true"></i> : Still one OS per machine. 
   
2. **No seperation of concerns** <i class="fa fa-check-circle text-success" aria-hidden="true"></i> : Atleast we can now trace user activity and ensure one user cannot modify files used by other users unless explicitly permitted to.
   
3. **Resource Utilization Inefficiency** <i class="fa fa-times-circle text-danger" aria-hidden="true"></i> : The machine's storage for example is still shared among all users


### Virtualization


Then along came virtualization. Virtualization adds a software called a Hypervisor that runs on your Operating System and emulates the hardware and the BIOS. This allows you to install multiple operating systems on this "virtualized" layer. With this you could run multiple operating systems, such as multiple versions of Windows, Linux etc. on the a single host machine. 

In layman speak, the hypervisor fools the guest operating system into believing that it has access to actual hardware. The hypervisor in turn becomes the layer in between that translates messages that need to pass to the actual hardware. So for example, in the guest operating system, you create a text file that has the contents 'hello' in it, the act of creating the file and writing into it is passed from the guest operating system to the host and then using the drivers on the host machine, to write into actual storage. This is obviously over simplified and doesn't work as slow as it sounds it would. There are obviously type 1 and type 2 hypervisors that may or may not require a full blown host operating system to be running to run the hypervisor on.

Think of it like inception... a dream within a dream running inside the same head!

1. **One OS per machine Limitation** <i class="fa fa-check-circle text-success" aria-hidden="true"></i> : Multiple guest operating systems per host machine. 
   
2. **No seperation of concerns** <i class="fa fa-check-circle text-success" aria-hidden="true"></i> : All applications can now run in their isolated Operating System environments. 
   
3. **Resource Utilization Inefficiency** <i class="fa fa-check-circle text-warning" aria-hidden="true"></i> : The machine's CPU, RAM and the Storage however are still shared across all the guest virtual machines.

While enabling running 10~20 VM's on the same physical hardware, virtualization came at it's own cost:
    
1. Every guest operating system takes up the same amount of storage as they would without virtualization. Eg: If the size of a single windows installation is 1 GB, including the size of the host operating system, 10 VM's would mean 10GB of storage.

2. Even when the Virtual Machines were running but not actively running an application inside them, they'd still be utilizing some RAM and CPU for the OS to perform it's usual base operations. Eg: Assuming simply running Windows on your machine without any applications (like right after startup) uses 1GB of RAM, running 10 Windows VM's would cost 11 GB of RAM in total.

3. Patching becomes a nightmare. If you had to install security updates for windows, adding multiple virtual machines increases the amount of effort that goes into patching all the machines.

<div class="mermaid text-center">
graph TB
    subgraph Flow
    Application-->o1[Guest Operating System]-->Hypervisor-->o2[Host Operating System]-->BIOS-->Hardware
    end
style Flow fill:#f1f1f1
style Flow stroke-width:0px
</div>

> A more detailed post on Virtualization and how we got to it coming soon..

### Containerization

Containerization has existed since the 1970's in it's early forms however with the inset of technologies such as Docker, much standardization was introduced in container technology. While VM's are an abstraction of the Hardware layer, containerization is abstraction on the Operating system level. Technically speaking, a way to think of it is similar to how you can have multiple users on a machine. If you login to the machine from 2 different users at the same time, while the users share the same operating system, they essesntially have thier own files and processes running for themselves wihtout one being able to look into the processes running for the other user. There are a couple of different namespaces (7) such as pid, cgroups etc. that let you do this isolation for different kinds of resources such as mount points, network, process PID's etc. but essentially, containers use the underlying namespaces in the host linux kernel to create processes under a PID (think PID inside PID) which are isolated from the other global PID's. Containers also use the other namespaces such as cgroups and networking namespace to ensure rosource utilization caps and ensuring that the forked processes can run exposing the same network ports and expose different ports on the actual host operating system (using port forwarding).

1. **One OS per machine Limitation** <i class="fa fa-check-circle text-success" aria-hidden="true"></i> : Multiple docker containers running different operating systems on the same host machine - using the base kernel of the host but differential binaries from the OS running inside the container.
   
2. **No seperation of concerns** <i class="fa fa-check-circle text-success" aria-hidden="true"></i> : All applications run in their own isolated container environment but do at a very base level share the same kernel. From an application perspective, there's no talking to the other containers via the container abstraction layer. 
   
3. **Resource Utilization Inefficiency** <i class="fa fa-check-circle text-success" aria-hidden="true"></i> : More efficient than containers as the basic kernel functionalities are reused form the host operating system, hence making the container's runtime footprint extremely light weight.

<div class="mermaid text-center">
graph TB
    subgraph Flow
    Application-->Container-->c1[Container Daemon on Host]-->BIOS-->Hardware
    end
style Flow fill:#f1f1f1
style Flow stroke-width:0px
</div>


### Kubernetes

Containerization was all the rage but it had it's own share of issues such as networking between containers (how do containers talk to each other), load balancing requests between containers, managing storage mount points and for the truly adventurous at heart - managing ports on the underlying server for containers. This led to the adoption of container orchestration platforms that simplify and abstract some of the issues with containers such as docker swarm, kubernetes etc. Kubernetes in that regard is an orchestration layer for containers that was contributed to the open source community laying the founding stones of what is today known as CNCF - the Cloud Native Computing Foundation, a vendor neutral home for many of the fastest growing open source projects.

Kubernetes, as an orchestration platform is built using Go (programming language) which generally speaking does the following:
1. Abstracts the underlying infrastructure by maintaining a pool of nodes (Virtual Machines or vanilla servers running an operating system aka baremetal).
2. Standardizes the deployment topology for containers (in a construct called a pod), the networking (ingress and egress), resource limits, replicas, distribution of pods, configuration, security keys etc. as simplistic yaml (Yet another markup language) files.
3. Simplifies the process of container orchestration across a node pool as well as the networking between them.
4. Provides the functionality that makes canary deployments, rapid CI/CD extremely easy to perform and manage.
5. Increases application horizontal scalability.

This is not to say that Kubernetes doesn't have its own share of gotchas, but Enterprises and startups are taking to the platform by storm helping maturity and large scale adoption. Basically though, even when you're using containers, you'd always need to have atleast a bare minimum number of containers running even when there's no load on the system, holding up infrastructure resources. 

What if you could consume the underlying infrastructure on a as and when required basis? Computing today is moving to an age where this too must be abstracted out. In comes Serverless..


### Serverless

Serverless as the name suggests isn't really a no server architecture. Instead serverless is an architecture that allows cloud providers such as Oracle, AWS, Google etc. to provide a commercial model that allows you to use multi-tenant storage, software defined networks (combined, known as Backend as a Service) and a network of compute nodes that can quickly spin up lightweight containers for computing (called Functions as a Service) while paying for all of this on a as per use basis.

1. Pay for Storage in terms of the GB of space you consume (think Cloud Object Storage)
2. Pay for Network on a per use basis (think data egress)
3. Pay for compute for the number of incoming messages. Don't pay when there are no messages

This virtually eliminates the need to provision infrastructure and pay for even when you're not using it, ushering an era of solutions that deliver blazing fast solutions, at scale, at minimal charges. 

But like we all know, with great power comes great responsibility and Serverless ain't no saint... but that's a post for another time!
