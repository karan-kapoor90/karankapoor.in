---
layout: post
categories: [Workshops]
comments: true
excerpt: A short and sweet getting started with Kubernetes guide.
title: Kubernetes Workshop
tags: [kubernetes, cloud native, orchestration, microservices]
description: A short and sweet getting started with Kubernetes guide.
---

This is in continuation from the [docker workshop blog](% post_url 2020-04-15-docker-workshop %). 

## Why the orchestration?

In the last blog post, we created docker containers and learnt that we could have multiples of those on the same machines or multiple machines, and have them provide High Availability for our applications. Sure, when you're talking about a few containers, that's feasible. But consider running 100's and 1000's of containers. Some of the basic issues you'd run into:
- Since containers are, by definition ephimeral, meaning that they should die when they've served their purpose, and can be started and stopped at will without impacting users, in such a case, how do you ensure that dead containers come up instantly to replace dead containers as required?
- Load balancing across 100's of containers running on every node, on the host's ports dynamically, on layer 7 would be extremely difficult to program.
- Networking b/w containers running on different nodes would require complicated networking fir inter container communication.
- Logically grouping containers from an application perspective would be near impossible.
- The number of containers that can run on a node would be limited by the number of ports available on the node. 

These are but very few of the reasons why an orchestration engine that coordinates containers and provides the functionality for them to work efficiently as a network of containers running services to power an entire application is required. 


## Prerequisites


- A running kubernetes cluster - this could be a local cluster, a docker desktop cluster, a cluster on some cloud provider etc.
- [kubectl cli](https://kubernetes.io/docs/tasks/tools/install-kubectl/) - the kubernetes Command Line Interface to interact with your k8s cluster using commands
- A little patience :)
- We'll be generating the kubernetes resource files ourselves in the following guide, but if you're not feeling adventurous you can [download all the resources]() we're going to be using.


> Note: The kubectl CLI may already be installed with docker desktop, but may not in your PATH. In windows it may be available at C:\>Program Files\Docker\Docker\Resources\bin\kubectl.exe 


### Verify access to your cluster


1. Try accessing your cluster and if the context is set appropriately

    ```bash
    $ kubectl get nodes   # this should respond with the list of available nodes.
    ```

2. A deployment is a kubernetes resource that orchestrates the replication of pods using a pod selection method based on the pods labels. So for example if you deploy pods with label color=red, and you want to ensure that certain number of replicas are maintained across the kubernetes cluster at all times, deployments can enable this.

    ```bash

    # Method 1: Simple using the CLI - less control

    $ kubectl create deployment deploy-workshop --image=karankapoor/docker-workshop:latest  # this would simply create a deployment with the mentioned image with replicas set to 1. 

    # Method 2: Using the declerative approach, through a yaml file
    
    $ kubectl create deployment deploy-workshop --image=karankapoor/docker-workshop:latest --dry-run -o yaml > deploy-workshop.yaml   # this will create the yaml resource definition file for a deployment using the mentioned image.
    ```
  

    - Method 2: Editing the yaml file

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        tier: web   # take note of this label
      name: workshop-deploy
    spec:
      replicas: 3   # increase the number of replicas to 3 - this is how many pods will be created
      selector:
        matchLabels:
          app: workshop-deploy
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: workshop-deploy
        spec:
          containers:
          - image: karankapoor/docker-workshop:latest
            name: docker-workshop
            imagePullPolicy: Never
            ports:   # Add this section
            - containerPort: 3000   # exposes port 3000 of the container 
            resources: {}
    ```

3. Deploy these resources onto the container

    ```bash
    kubectl get deploy   # notice no deployments
    kubectl create -f deploy-workshop.yaml
    kubectl get deploy
    ```

    Notice the pods created by the deployment. There should be 3 pods.

    ```bash
    kubectl get po
    ```

4. Expose all 3 pod instances containing the docker-workshop container on 1 port on the host machine using a NodePort Service resource on the cluster. Let's use the declarative approach - the smart way

    ```bash
    kubectl expose deployment workshop-deploy --type=NodePort --port=3000 --target-port=3000 --dry-run -o yaml > service-workshop.yaml
    ```

    The added advantage of using this mechanism to expose the deployment is that it will automatically copy the labels from the deployment into the backend selection criteria in the service selector. The service by default in this manner would share the same name as the deployment, but you can change it in the yaml if you like.

5. Deploy the service to the cluster

    ```bash
    kubectl create -f service-workshop.yaml 
    ```

6. Get the NodePort on which the service is exposing all the pods created by the deployment. 

    ```bash
    kubectl describe service workshop-deploy 
    ```

7. In the response of the describe command, look at the nodePort and the endpoints section. The endpoint mentions the Cluster internal IP address of the pods where the service is running, while the nodeport mentions the port on which the service is exposed on the host machine.


And voila! In about 15 minutes, you've gone from a docker container to multiple replicas of the container running immutably across a cluster of nodes. Even if any of the pods or even the nodes die, kubernetes will ensure that your application is always running!! 

> Bonus, you can scale the number of backend pods up or down with a single command

```bash
kubectl scale deployment workshop-deploy --replicas=5
```

There's plenty more to do, 
- Persisting storage shared across the pods
- Service discovery 
- Load Balancing
- Ingress and Egress
etc. 

Watch this space for more :)