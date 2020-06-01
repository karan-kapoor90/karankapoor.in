---
layout: post
title: k8s Tiny Tales - Creating Baby Admins
categories: [k8s Tiny Tales]
comments: true
tags: [kubernetes, cloud native]
excerpt: Create more admin users for your cluster namespaces
image:
  feature: blog-tiny-tales.png
---


## Background

Sometimes you're not the center of the world. Sometimes, you're replaceable! Just kidding. You may still be needed. 

I had a hell of a time while trying to add admin users to my kubernetes cluster, who could only be the admin for a particular namespace. Since they don't have access to the whole cluster (cus then I'd be jobless), I call them baby admins. 

## Overview

Here's a long and short of what the process looks like:

1. A new kid called John joins my team. I think he's bright, and he's cleared his CKA, so I think it's appropriate I let him administer a light weight namespace. So
    - John creates a private key for himself using openssl.
    - He then creates a Certificate Signing Request and shares it with me.

2. I've gotta do my job right, so I open up his certificate signing request and check to ensure he's not added himself to any inappropriate groups.
3. I use the CertificateSigningRequest resource in kubernetes to create a signing request and push it to the cluster.
4. Because I'm the admin, I then approve the request.
5. The extract the certificate from the approved certificate signing request inside my cluster and share it with the user along with the appropriate information he requires to setup a kubernetes config file for himself.
6. Additionally I need to optionally create the role/ use an existing role to create a role binding that binds his username to the role on the cluster.
6. John uses the files I supplied to create his own kube config file and enjoys baby god mode. 

## John initiates the story

Create a private certificate for himself

```bash
$ openssl gensra -out john.key 2048
```

Then he creates a certificate signing request to share with me.

```bash 
$ openssl req -new -key john.key -subj "/CN=john" -o john.csr
```

He then shares the `john.csr` file with me. The `john.key` file is his to keep... his little secret.

## Admin receives the csr

I use the csr John submitted and check it's subject name for correctness.

```bash
$ openssl req -in john.csr -text -noout
```

Then I use this to create a CertificateSigningRequest resource on the k8s cluster. 

> I tend to get a little lost with the API version etc of the CertificateSigningRequest, so I choose to check it as follows `kubectl explain csr`

I could use some terminal magic to convert and put in the file in one short ( ;) yeah right ), but the regular way, convert the john.csr, the certificate signing request John submitted to base64 and copy it.

```bash
$ cat john.csr | base64
```

Then I create a yaml file as follows:

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: john-admin   # Note, this is the name of the certificate/ csr resource we're about to create
spec:
  usages:
  - client auth
  - key encipherment
  - digital signature
  groups:
  - system:authenticated
  requests: #base64 value of john.csr - simply pasted here
```

Apply this yaml to your cluster. This is not a namespaced object so it doesn't matter which namespace you're applying this file into.

```bash
$ kubectl create -f csr.yaml
```

I can view the csr I created using the following command

```bash
$ kubectl get csr

# and then approve it using the following command

$ kubectl certificate approve john-admin
```

Once this is done, I have effectively permitted John's private key to be authenticated against the cluster, meaning that at this point, when I share the certificate with John, he'll be able to connect to the cluster. But since we haven't authorized him yet, he won't actually be able to do anything against the cluster.

I take out the certificate that I will now share with John as follows

```bash
$ kubectl get csr john-admin -o=jsonpath={.status.certificate} | base64 --decode > john.crt

# The status, after approval contains the certificate that I will share with John. Inside the certifocate object it is base64 encoded, so I need to decode it before sharing it with John. I also write it out to a crt file. 
```

In order to authorize John's user, I can now either give him cluster wide permissions, or namespaced access using ClusterRoleBindings or RoleBindings respectively.

Let's assume the namespace that I want to give John access to is called 'john'. Using imperative commands, I can now use the following command to setup the namespace specific RoleBinding as follows:

```bash
$ kubectl create rolebinding john-admin --clusterrole admin --user john --namespace john
```

> At this point, what stops me from assuming John's persona and logging into the cluster using his user is his private key that he held back from me in the first step :). He's a smart boy!

At this point, the role binding has associated John with the admin clusterrole, but limited to the namespace john. Now all that John needs is the files to make a kubeconfig file for himself. For that, I need to share the following files with him:

- Cluster Information
    - Cluster CA certificate
    - API server URL
- User information
    - The crt certificate I generated for John

Here's how I get the information

### Cluster CA certificate

I can pull this from my config as follows:

```bash
$ kubectl config view -o=jsonpath={.clusters[0].cluster.certificate-authority-data} --raw | base64 --decode > ca.crt
```

### Cluster Server name

```bash
$ kubectl config view -o=jsonpath={.clusters[0].cluster.server} > server-address.txt
```

I share the following files with John:
- john.crt
- ca.crt
- server-addess.txt


## John sets up his kube config

John now has a couple of things to do, to consume the files I shared with him and create his kubeconfig file. Here's what he does.

```bash

# setup the cluster details in a config file called dev-config
$ kubectl config set-cluster dev-cluster --server=$(cat server-address.txt) --certificate-authority=ca.crt --kubeconfig=dev-config --embed-certs

# setup the user details in the config file called dev-config
$ kubectl config set-credentials john --client-certificate=john.crt --client-key=john.key --kubeconfig=dev-config --embed-certs

# setup the context to use the cluster and the user we just setup
$ kubectl config set-context john@dev-cluster --user=john --cluster=dev-cluster --namespace=john
```

At this point, John can start using the kubeconfig file by

```bash
#  setting it in the KUBECONFIG variable 
$ export KUBECONFIG=$KUBECONFIG:$(pwd)/dev-config

# switching the context to use the john@dev-cluster context in the john namespace
$ kubectl config use-context john@dev-cluster
```

Now, if John tried to create new resources in any other namespaces, he'd be denied, but would be the king of his own namespace :-)


Stay tuned for more kubernetes tiny tales.