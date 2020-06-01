---
layout: post
title: k8s Tiny Tales - Rollouts are underrated
categories: [k8s Tiny Tales]
comments: true
tags: [kubernetes, cloud native]
excerpt: Creating and working with rollouts in kubernetes
image:
  feature: blog-tiny-tales.png
---


## Background

While the basic unit of work in kubernetes is a Pod, which may contain one or more containers, when truly running apps on kubernetes, one tends to use deployments.

If you're a k8s beginner and you're beginning to itch wanting to know why, don't fret my friend.

- Deployments are an abstraction on top of something else known as ReplicaSets. As the name suggests, this is the resource type that actually ensures that the number of replicas of a pod running in your cluster is maintained. Hey, but then why deployments and not just replica sets?

- Deployments manage replica sets such that every time you make changes to it, it creates a new replicaset and depending on the rollout strategy mentioned in the deployment's configuration, it scales down the old replica set and brings up the new one.

- Also, there is the small but significant technicality of how a replicaSet or a deployment identify which pods they're controlling. ReplicaSets allow you to match labels as the pod. However, deployments allow you to match expressions as well as labels to select which pods it oversees. 

But all this is nice and good. The real reason why we need deployments is because replica sets lack the ease with which one could do rolling updates with replication controllers (predecessor to replica sets). Hence deployments!

And voila, this functionality of being able to do rolling updates, trace them back in history and rollback to an old state is what deployments bring to the table using the `rollout` command. Let's see it in action.

## The Story

John, my baby admin from the [previous post](k8s-tiny-tales-creating-baby-admins), is the assigned kubernetes operator for a team of devs who can go so far as to create docker containers of their apps, but know nothing of deploying them on kubernetes. So John [containerises their app](kubernetes-workshop). 

Over time, he's helped them release newer version of their app. Somewhere down the road though, the team got complacent and released some extremely broken software and John is tasked with rolling the code back with no knowledge whatsoever of the code itself. So this is what happens:

- John deploys an app to the cluster
- John releases a new version of the app
- John releases another new version of the app
- John gets a call at 2 AM in the night and must rollback the code


## The actual action

### Deploying an app to the cluster:

John can create a simple deployment to the server as follows:

```bash
$ kubectl create deployment team-app --image nginx:1.16

# Scales the app to 10 replicas as the desired state 

$ kubectl scale deployment team-app --replicas=10 --record
```

and check the status of the app rollout as follows.

```bash
$ kubectl rollout status deployment team-app 
```

### Releasing a new version of the app

When the team confirms they've created a new version of the app for example nginx:1.17, here's what John can simply do

```bash
$ kubectl set image deployment team-app nginx=nginx:1.17 --record  # Here nginx=nginx1.17 means change the image for the container name 'nginx' with image 'nginx:1.17'
```

and once again John checks the status, as the change goes through to all the 10 running pods, as follows:

```bash
$ kubectl rollout status deploy team-app
```

### Releasing yet another version of the app


```bash
$ kubectl set image deployment team-app nginx=nginx:1.18 --record  
```

and once again John checks the status, as the change goes through to all the 10 running pods, as follows:

```bash
$ kubectl rollout status deploy team-app
```

### And then the dreaded 2 AM call

Right out of deep sleep, awoken by a call the same night as the latest deployment, the users have complained their payments are failing, an issue that the team has diagnosed as something that's gone as a part fo the latest release. While the team reworks the app and fixes the issue for good, John must save the day by rolling back to the nginx:1.17 image. But he needs to keep track of what was done on the cluster as well.

Here's what he does:

```bash
$ kubectl rollout history deploy team-app

##### OUTPUT ######
deployment.apps/team-app 
REVISION  CHANGE-CAUSE
1         kubectl scale deployment team-app --replicas=10 --record=true
2         kubectl set image deployment team-app nginx=nginx:1.17 --record=true
3         kubectl set image deployment team-app nginx=nginx:1.18 --record=true

```

He realizes that he needs to now rollback to revision 2, so here's what he does:

```bash
$ kubectl rollout undo deploy team-app --to-revision 2 --dry-run  # As the command option --dry-run suggests, this will show him exactly what will be applied to the cluster once he actually runs the undo

##### OUTPUT #####
deployment.apps/team-app Pod Template:
  Labels:	app=team-app
	pod-template-hash=7cbfdf4ff9
  Containers:
   nginx:
    Image:	nginx:1.17
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
 (dry run)

 $ kubectl rollout undo deploy team-app --to-revision 2   # rolls back the deployment to the state as of revision #2

 $ kubectl rollout history deploy team-app   # Notice that the revision 2 stops showing in the history and now shows up as revision #4 in the history
```

And just like that, John saves the day! 

### The Pledge

This 2AM call made John realize that just like the git revisions of the code that the team has, the deployment should also be automated. Ideally,

- A rollback (revert in git speak), of the code should trigger a rollback of the associated kubernetes deployment as well
- the removal of revision #2 from the rollout history isn't an ideal situation. Ideally, a new revision should have been created with the same contents as revision #2, and applied to the cluster. Git style!

This git style of doing things is also called Git-ops.

> John pledges to setup Continuous Integration and Continuous Deployment with a GitOps methodology to automate code deployment.

Stay tuned for more.