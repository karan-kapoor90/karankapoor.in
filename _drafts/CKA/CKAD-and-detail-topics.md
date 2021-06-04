---
layout: post
categories: [Writeups, Kubernetes]
comments: true
excerpt: Open topics to study 
title: Topics to Study
tags: [kubernetes, cloud native]
description: 
---

# Topics to study

## Multi Container Pod Patters

There are 3 multi container pod patters:

1. Sidecar

Sidecar pattern is when you put another pod such as a logging agent is placed in the same pod beside say your web-server pod. 

2. Adapter

An example of the adapter pattern is when say a container is deployed besides the main container that reformats the logs from the main container before sending them out to say a common aggregator layer.

3. Ambassador

All apps typically run the issue of having to change the target endpoint for say, a service, the ambassador pattern acts like a proxy between the app code and the other service, directing traffic accordingly.

# Additional k8s topics

## Liveness Probes

## Readiness Probes

