---
layout: post
title: k8s Tiny Tales - Hold your secrets close
categories: [k8s Tiny Tales]
comments: true
tags: [kubernetes, cloud native, secrets]
excerpt: Secret's aren't all that secret in k8s, and there are multiple ways to read secrets deployed in a cluster, given you have access.
image:
  feature: blog-tiny-tales.png
---

## What are secrets?

More often than not, your applications would need to connect to some external system like a database, a messaging queue, some SaaS tools etc. These external tools would need authentication or some sort - token, username/ password, certificate etc. 

Kubernetes has a resource called configMaps which allows you to have these configurations as key-value pairs. But these configMaps are plain text and can be read anyone. So, k8s came up with another resource called secrets which essentially do the same job, except they're base64 encoded. 

That obviously isn't as secure. 

## Show me your secrets 

> "I got a little secret for you. Come here. No, closer. I don't make deals with peasants!" - From the movie 'Emperor's new Groove'

In order to read secrets, you need to have the correct roles and permissions in the namespace (or cluster wide) to be able to do so. Which, in case you already have, you're a couple of commands away from reading them secrets anyways - they're simply base64 encoded data - remember?

So this post is about a few ways in which you can do that :-)

At the time of this writing, there are 3 types of secrets that you can create using the kubernetes API:
- generic 
- docker-registry
- tls

### Pre-req

I like using a nice little tool called `yq` which is like  a `jq` or `sed` for yaml files, that can be used to query through yaml documents rather neatly. You can download it here: [Mike Farah's yq github repo](https://github.com/mikefarah/yq)


### Common bit - Finding what you're looking for

```bash
$ kubectl get secrets       # Lists all secrets
$ kubectl get secret <secret-name> -o yaml      # Select the name of the secret you want to open up

# Here, you must look at the field names inside the data key of the secret. The values of the sub-fields under data are what are base64 encoded. For example you could have field names like .dockerconfigjson, username, password, token etc. depending on the kind of secret created.  

```

Example: Let's assume you're trying to crack open a secret called github-creds with username and password in them.

```bash
$ kubectl get secrets

NAME                  TYPE                                  DATA   AGE
default-token-k5g8j   kubernetes.io/service-account-token   3      93s
github-creds          Opaque                                2      18s

$ kubectl get secret github-creds -o yaml

apiVersion: v1
data:
  password: T2J2aW91c2x5Tm90TXlQYXNzd29yZA==
  username: T2J2aW91c2x5Tm90TXlVc2VybmFtZQ==
kind: Secret
metadata:
  creationTimestamp: "2020-10-20T07:25:39Z"
  name: github-creds
  namespace: bs
  resourceVersion: "11464930"
  selfLink: /api/v1/namespaces/bs/secrets/github-creds
  uid: ca8b5074-718f-4496-85b5-9db94cd60ad0
type: Opaque

```


### 1. Manually Copy pasting and getting cracking the data

Copy the base64 encoded data using your cursor and put it into the following command

```bash
$ echo T2J2aW91c2x5Tm90TXlVc2VybmFtZQ== | base64 --decode       # decoding the username
ObviouslyNotMyUsername      # Output

$ echo T2J2aW91c2x5Tm90TXlQYXNzd29yZA== | base64 --decode       # decoding the password
ObviouslyNotMyPassword      # Output

```


### 2. Using yq and decoding

```bash
$ kubectl get secret github-creds -o yaml | yq r - data.username | base64 --decode
ObviouslyNotMyUsername      # Output

$ kubectl get secret github-creds -o yaml | yq r - data.password | base64 --decode
ObviouslyNotMyPassword      # Output
```


### 3. Consuming inside another pod

> Admittedly you'd probably never be using it if all you want is to view secrets, but I thought this was cool if you wanted to so some tomfoolery with someone else's database credentials :-) 

Create a busybox pod --> mount the secrets as a volume --> access the code inside the container

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - image: busybox
    name: busybox
    command: ["sleep"]
    args: ["60000"]
    resources: {}
    volumeMounts:
    - name: creds       # mounting the volume creds into this specific container in the pod
      mountPath: '/etc/creds'       # path inside the container where the creds will be present
      readOnly: false
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: creds      
    secret:
      secretName: github-creds      # mounting the github-creds secret as a volume on the pod
```

And then sh into the container and `cat /etc/creds/username` or run the following command directly:

```bash
$ kubectl exec busybox -c busybox -i -t -- cat /etc/creds/username      # run the cat command inside the container and print the output
ObviouslyNotMyUsername      # Output

$ kubectl exec busybox -c busybox -i -t -- cat /etc/creds/password      
ObviouslyNotMyPassword

```

Personally, I like using approach #2. 


## A solution

The good news is, that besides the [best practices mentioned in the official kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/secret/#best-practices), there are alternative solutions out there [like hashicorp vault](https://www.vaultproject.io/) ([Example usage](https://www.hashicorp.com/blog/injecting-vault-secrets-into-kubernetes-pods-via-a-sidecar)) and [Kamus](https://github.com/Soluto/kamus#:~:text=Kamus%20enable%20users%20to%20easily,the%20blog%20post%20and%20slides.).

Do you use some other ways to do this? Comment and tell me how!