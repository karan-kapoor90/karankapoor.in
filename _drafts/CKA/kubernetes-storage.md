---
layout: post
categories: [Writeups, Kubernetes]
comments: true
excerpt: A guide for administering and navigating storage in Kubernetes
title: Kubernetes Storage
tags: [kubernetes, storage, persistence]
description: A guide for administering and navigating storage in Kubernetes
---

## Storage Basics for Containers

### Basics

When you install Docker, it creates the `/var/lib/docker` directory, which has the 3 following folders inside it:

- aufs
- containers
- images
- volumes

Because docker uses a layered file structure, such that every command in a dockerfile creates a new image, all the data on the image is Read-only. When you create a container using the `docker run` command, it creates a new read-write layer on top which can have all your app logs etc inside. This is the data that goes inside the containers folder. 

Your app will usually also be a part of the image, in the step where you copy it into the container in the Dockerfile. When you open a terminal session inside a running container and try to modify your application code, docker uses a copy on write mechanism to create a copy of the file from the image layer into the container storage layer and then making modifications. Thus all the modifications you make to the application code as well are ephemeral and live only as long as the container itself. 

#### Volume Mounting

In the case of wanting to persist the data created inside a container, such as app transaction logs or DB files, we use Volumes. When you create a volume using the `docker volume create myVolume` command, it creates a directory in the volumes folder with the same name as the volume you created.

We then mount this volume into a container when we run it as `docker run -v myVolume:/var/lib/data <image-name>`.

> Even if you don't explicitly run the docker volume create command before trying to mount it on a container, running docker run and mounting a volume will automatically create the volume as well.

Example: `docker run -v myVolume:/var/lib/appData`

#### Bind Mounting

In the case that you already have data in a folder and just wish to make it available inside the container, you use the same `docker run` command with the `-v` option to mount the folder. You give the full path of the folder you want to mount on the host. Hence the host directory can be any location you specify.

Example: `docker run -v /host/folder/location:/var/lib/appData`

> Volume mounting mounts a particular volume from the volume directory to the container, while the bind mount mounts a particular directory from the host machine.

The newer notation is using the `--mount` option instead of `-v` because it is more verbose.

- Volume Mount: `docker run --mount type=volume,source=myVolume,target=/var/lib/appData mysql`
- Bind Mount: `docker run --mount type=bind,source=/host/folder/location,target=/var/lib/appData mysql`

### Storage Drivers

Docker uses storage drivers to make the writeable layers, creating the copy on write storage etc. Various drivers are available, which depends on the OS you're running on:
- AUFS - for Ubuntu
- ZFS
- BTRFS
- Device Mapper
- Overlay
- Overlay2

Docker automatically chooses a Storage driver. Different drivers may give different performance etc. 

### Volume Drivers

Volumes are handled by volume driver plugins. The default plugin is called Local. Azure File Storage, Convoy, Digital Ocean Block Storage, GlusterFS, VMware vSphere storage etc. These drivers can support multiple different storages. When creating a docker container, we can use the `--volume-driver` option to specify the volume driver to use.

## Container Storage Interface

CRI is Container Runtime interface - this is the set of properties that any container Runtime provider must implement.

CSI - is not a k8s specific standard and is meant to allow any container runtime interface with different Storages. 

- Ceph
- GlusterFS
- NFS
- Flocker
- AWS EBS

Must implement:
 - Orchestrator should be able to call the RPC to create volume and the volume driver should be able to provision it. Volume should have a name.
 - Orchestrator should be able to call delete volume RPC and the volume driver should be able to decommision it.
 - Orchestrator should be able to make the call to place a workloads that uses the volume on the node. The driver should make the volume available on the node.

 ### Simple directory mount on a single node cluster

 A single node cluster may just need a folder to be mounted on to a pod as a volume. This should not be done for a multi-node cluster as it'll expect all the nodes to have the same folder, which will not be the case. 

 ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: alpine
    command: ["/bin/sh","-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/random-number.out;"]
    volumeMounts:   # since the container can have multiple mounts, volume mounts is an array
    - mountPath: /opt
      name: data-volume

  volumes:   # multiple volumes can be defined for getting consumed by a single or more containers inside a pod
  - name: data-volume
    hostPath:   # hostpath means that the volume being loaded is a path from the host machinec
      path: /data
      type: Directory
 ```

### Persistent Volumes on a Multi-node Cluster

In the case of a multi-node cluster, the pod may be created on any of the nodes, and the data must persist on the storage while seeming the same to all the pods on the cluster. Ideally, the admin would want to create a single large pool such that consumers can then cut out a portion for them and use it for their pods. Also the configuration and management of these volume mounts should be simpler, in the sense that if they were defined in the pod definition file like earlier, making changes would mean modifying all the pod definitions individually which would be extremely tedious.

Volumes can be mounted in one of 3 modes:
- ReadWriteOnce
- ReadWriteMany
- ReadOnlyMany

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: 
  name: pv-vol1
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:   # mention hostpath for the path on the host node. Never use in production.
    path: /tmp/data
  persistentVolumeReclaimPolicy: Retain
```

If you're deploying this on AWS, replace `hostPath` with the cloud provider provided volume driver:

```yaml
awsElasticBlockStorage:
  volumeID: <volume-ID>
  fstype: ext4
```

#### PV Claims

Volumes must be claimed in order for them to be used in pods.
Admins create set of persistent volumes and the users create PV Claims to use them. k8s binds the claims basis set of properties, and binds 01 volume to a 01 claim. There is a 1:1 relationship b/w volumes and claims

If there are multiple available matches then you can use selectors and labels to match a claim to a particular volume as well. Criteria for selecting a volume for a claim depends on 
- Sufficient Space
- Access mode
- Volume Mode
- Storage Class
- Selector

> A smaller claim may bind to a larger volume, but even then, no other claim can utilize the volume, because the relationship is 1:1.

If a PV is not available to meet a PVC, the PVC stays in a pending state till one is made available.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

When you delete the claim, what happens to the underlying volume can be controlled with the `persistentVolumeReclaimPolicy`. Possible options are:
- Retain: PV will remain till manually deleted, cannot be used by any other claim
- Recycle: The data in the data volume will be scrubbed before making it available to other claims
- Delete: As soon as the claim is deleted, the volume will also be deleted

Using it in a pod definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: nginx
    volumeMounts:
    - mountPath: "/var/www/html"
      name: myvol
  volumes:
    - name: myvol
      persistentVolumeClaim:
        claimName: myvol
```