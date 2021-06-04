---
layout: post
title: k8s Tiny Tales - Opening other's secrets
categories: [k8s Tiny Tales]
comments: true
tags: [kubernetes, cloud native]
excerpt: Reading secrets being used by other/ running pods
image:
  feature: blog-tiny-tales.png
---


## Background

> Your secret's aren't really that secret!

Let me explain. Secrets in Kubernetes, out of the box, aren't really particularly secretive. But what is a secret, and why would I need it, you ask? 

- Secrets are an Out of the box resource in Kubernetes. Just like your pods, deployments, ConfigMaps, services etc. 
- Secrets are what you'd create to hold information such as certificates or username/ password or tokens etc. that may be used by your code. 

Here's the official documentation - https://kubernetes.io/docs/concepts/configuration/secret/

## The Story

Let's say I get access to a cluster with some services already running on it, assume a redis service. In doing so, whoever created it, must have had to pass admin 


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