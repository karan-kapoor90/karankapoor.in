---
layout: post
title: Path based routing
categories: [Tutorials]
comments: true
tags: [kubernetes, cloud native, ingress, cloud native]
excerpt: Setup ingress using nginx-ingress controller on kubernetes
---

## Background

Let's face it, when we're doing micro-services, not every service is going to warrant the cost of a software load balancer fronting every individual service. Besides you'd typically segment your apps deployed on k8s using one of the two following strategies:

- Create subdomains for individual apps, such as blog.mysite.com or code.mysite.com. Here, `blog` and `code` are called sub-domains - meaning domains that are children to your base domain `mysite.com`

- Create application routes such as mysite.com/blog and mysite.com/code, where `blog` and `code` are routes under the base domain. 

> Obviously, your links could be a combination of both the strategies such as a `code.mysite.com/live` 

## Kubernetes

The various mechanisms available in k8s to expose your apps to the outside world, employ services. The particular services that can help you do so would be `NodePort`, `LoadBalancer` or `Ingress`. Read more about the available service types [here](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types). As a summary:

- `ClusterIP` - This is the default service type in kubernetes. This basically means that the service is allocated an IP address internal to the cluster. This will be used by other services (or pods) deployed on the cluster to talk to the service. This is strictly internal to the cluster, and cannot be seen by consumers outside the cluster.

- `NodePort` - NodePort type service uses the clusterIP service internally to allocate a kubernetes cluster private IP address which kubernetes then allocates a operating system port on all the worker nodes. This port is kept the same across worker nodes meaning that any external consumer who wants to consumer a service can call it on any worker node on that port. 

- `LoadBalancer` - As one can imagine, the idea of consumers outside the cluster knowing the IP addresses of all the worker nodes in the cluster sounds rather counter intuitive. The need to remember the port is an additional counter intuitive pain. Also, if the port is available on every worker node, why not just load balance across all those worker nodes and publish one single IP address and let all consumers use that single IP address. Additionally it'd be awesome to standardize the service on ports on 80 and 443 to external consumers (the default ports for http and https web traffic on the public internet) on this load balancer, and then assign it an awesome DNS name like `karankapoor.in` like the one you're at right now!! Hell ya!

And there comes the problem. A Load balancer implemented by cloud providers work such that, the moment you create a kubernetes service of type LoadBalancer, a cloud provider version of the load balancer is created and connected to the service on the cluster. As you can imagine, in a kubernetes world where we're trying to create microservices, 100's and 1000's of them, LoadBalancer's could turn out to be quite an expensive proposition. One could, painstakingly configure the load balancer for every individual service on a per NodePort basis, but that would be so so so so soooo painful. 

`Ingress` to the rescue - Ingress, as per the official Kubernetes documentation (as of 1.18) is not an actual service type, but instead acts as an entry point to your cluster. Officially:

> It lets you consolidate your routing rules into a single resource as it can expose multiple services under the same IP address. 


## So how do we do this?

In order to do ingress in your cluster, an ingress controller needs to be installed on the cluster. There's plenty of content available online on when to choose which and why, but the most popular of them is nginx-ingress controller. Confusingly there are 2 version by fairly similar names:

- kubernetes ingress-nginx - This is a part of the kubernetes OSS (Open Source Software) and is a fairly mature product. Learn more [here](https://github.com/kubernetes/ingress-nginx).

- NGINX Ingress Controller - this is an offering from Nginx (a part of F5) which comes in both an OSS as well as an Enterprise variant. Learn more [here](https://github.com/nginxinc/kubernetes-ingress).

> The differences, a subject for possibly another post, can be found [here](https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/nginx-ingress-controllers.md).


## Let's get down

For the remainder of this tutorial, we'll use the kubernetes ingress-nginx OSS. Functionally speaking, here's what happens:

- nginx creates a bunch of resources on your cluster such as roles, bindings, clusterroles, clusterrolebindings, service, deployments and the certificates required by these components to work

- if the underlying infra is a cloud provider, using additional configuration, it creates a load balancer on the cloud provider and ties that to the ingress controller

- if the underlying infra is a local baremetal machines/ VM, the external IP and loadbalancer IP needs to be supplied by you. This can be done by using an HAProxy or a MetalLB loadbalancer. 

> In this case, since we're probably doing this on your own local minikube or docker desktop kubernetes cluster, your machine acts as the load balancer. 

## Setting up

Use the setup on the official website to install the ingress controller resources on your cluster. In this case, when running on docker desktop (I'm using docker for mac), run the following command:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.0/deploy/static/provider/cloud/deploy.yaml
```

At the end of this, you should have a new namespace called `ingress-nginx` with a few finished pods and jobs, 2 services and one running pod by the name of `ingress-nginx-controller`. One thing to notice here

```bash
$ kubectl get svc -n ingress-nginx

NAME                                         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.104.118.199   localhost     80:31088/TCP,443:31761/TCP   28m
service/ingress-nginx-controller-admission   ClusterIP      10.105.51.155    <none>        443/TCP                      28m

```

The `ingress-nginx-controller` shows the external IP which is where the ingress controller will be exposing services. Here, because I'm running the whole setup on my local machine, the external IP is..... (drumroll) `localhost`.


## Running an app

We're all mavericks here right? I'm sure you are!! So you could bring your own app, or use the one I'm just about to supply. Here's what we're going to be doing. 

- We'll create a deployment which runs 3 copies of a pod.
- Expose the deployment as a clusterIP service. This service is local to the cluster and nobody from the outside can use this. 

Let's get into this:

```bash
$ kubectl create deployment mydeployment --image karankapoor/docker-workshop   # creates a deployment with one pod

# View the fruits of your labor

$ kubectl get deploy   # this will show my deployment

NAME             READY   UP-TO-DATE   AVAILABLE   AGE
mydeployment     1/1     1            1           14s

$ kubectl get pods   # see that pods are created. Note that your pods should be in Running state

NAME                              READY   STATUS    RESTARTS   AGE
mydeployment-6b6f9697dd-qzfcw     1/1     Running   0          80s

$ kubectl scale deploy mydeployment --replicas=3   # scale up your deployment to 3 pods. This will give you 3 pods of mydeployment

# Expose the deployemnt (meaning load balancing across the 3 pods) as a clusterIP service

$ kubectl expose deployment mydeployment --port 3000   # service created that's reachable inside the cluster at port 3000

# Verify that the service is created

$ kubectl get service

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
mydeployment         ClusterIP   10.103.163.167   <none>        3000/TCP   2m20s

# and validate that the service, deployment and pods are all behaving well with each other

$ kubectl port-forward svc/mydeployment 30000:3000   # maps your local machine's port 30000 to 3000 of the service.

# Now fire up a browser and navigate to http://localhost:30000 and you should see a index page

```

## Ingress the world

Now let's try and create a basic ingress resource to front the service. 

Create a file called mydeployment-ingress.yaml, and write the following contents into it.

```bash
$ vi mydeployment-ingress.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: / 
  name: mydeployment-ing
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: mydeployment
          servicePort: 3000
        path: /banana
    # host: kubernetes.docker.internal
```

A couple of things before we proceed.

Notice the 

- path which says `/banana`. This means that I want to redirect all requests coming to at `localhost/banana` to `mydeployment` service on the service port 3000.
- annotation `nginx.ingress.kubernetes.io/rewrite-target`. This signifies that any path that all requests coming to `/banana` should instead be redirected to `/` path of the underlying service. 

A combination of `/path` and `rewrite-target` allow you to prefix your API url with things like `/api/v1` etc. without making any changes to the actual code. 

Now let's go ahead and create this on our cluster

```bash
$ kubectl create -f mydeployment-ingress.yaml   

$ kubectl get ingress   # this shows the actual ingress resource that's created

NAME               HOSTS   ADDRESS     PORTS   AGE
mydeployment-ing   *       localhost   80      19m
```

As you can see, the address is `localhost` and the port is `80` -- provided you didn't already have something else running on port 80, in which case this will fail. 

Head over to your browser and open `localhost/banana` to be greeted by the same service we'd created earlier. 

Stay tuned for more!