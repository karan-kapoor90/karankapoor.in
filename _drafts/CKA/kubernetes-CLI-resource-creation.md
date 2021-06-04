---
layout: post
categories: [Writeups, Kubernetes]
comments: true
excerpt: A quick and dirty guide to create kubernetes resources directly from the CLI instead of using a yaml or JSON file.
title: Kubernetes CLI Resource Creation
tags: [kubernetes, cloud native, kubectl]
description: A quick and dirty guide to create kubernetes resources directly from the CLI instead of using a yaml or JSON file.
---

## Some Useful extensions to commands

Before diving into CLI commands to create specific k8s resources, some of the following options are common to all commands and can be extemely useful.

### Dry run the creation, don't actually create

--dry-run : will do a dry run and show what will be created. This cna help with debugging a command before actually running the command. 

-o yaml : stdout details of the resource that is going to be created (also works for get requests) in the form of a yaml file. Can be used along with --dry-run to see the yaml file file that will beused for creating the resource without actually creating a resource

-o yaml > somefile.yaml : Same as the previous command, simply writes the output to 'somefile.yaml'. Use wil --dry-run can be extremely useful to get started with writing a yaml file for a resources instead of typing it out from scratch.

-o=jsonpath={.spec.containers[*].image} : to get images of all containers inside a pod

--show-labels : show labels when getting resources

## Pod

```bash
$ kubectl run --generator=run-pod/v1 <pod_name> --image=<image_name> --labels="<key>=<value>"
# or
$ kubectl run <pod-name> --image=<imagename> --generator=pod-run/v1

```

A pod can be exposed outside on a host port by port forwarding as follows:

```bash
$ kubectl port-forward deploy/pod/replicaset/service <resource-name> <host-port>:<resource-port>

# for example

$ kubectl port-forward pod my-nginx-pod 8080:80
```

## Deployment

```bash
$ kubectl create deployment --image=<container_image> <deployment_name>     # Currently this doesn't allow you to define the number of replicas in the command itself.

$ kubectl scale deployment/<deployment_name> --replicas=<desired_number of replicas>        # This does not edit the actual yaml file for the deployment, hence copying the file to different environment would produce the original number of pods. A better way to do this is to use kubectl edit to edit the deployment file.
```

### Run command for creating pods and deployment

The run command can be used to create pods, jobs or deployments with a simple change in the imperative declaration, as follows:

```bash
# Specifying restart=Never creates a Pod
$ kubectl run my-app --image=nginx --restart=Never

# Specifying restart=OnFailure creates a Job
$ kubectl run my-app --image=nginx --restart=OnFailure

# Specifying restart=Always creates a deployment
$ kubectl run my-app --image=nginx --restart=Always
```

> The --expose switch can be used to expose a resource using a service

## Secret 

```bash
$ kubectl create secret generic <secret-name> --from-literal=<key-name>=value --from-literal=<key-name>=<value>
```

## ConfigMap

```bash
# from the CLI
$ kubectl create configMap <config-name> --from-literal=<key-name>=<value> --from-literal=<key-name>=<value>

# using a properties file
$ kubectl create configMap <config-name> --from-file=configFile.properties
```

