---
layout: post
categories: [Writeups, Kubernetes]
comments: true
excerpt: A guide to upgrading and maintaining your kubernetes clusters
title: Kubernetes Cluster Management
tags: [kubernetes, cloud native]
description: A guide to upgrading and maintaining your kubernetes clusters
---


# Worker Node OS upgrades

If you create and deploy pods, and the node on which they're running goes down, k8s waits for a default timeout of 5 mins for the nodes to come back up. If it doesn't come back up, the pods are terminated and k8s considers them dead. If the pods are a part of a replica set thne they're recreated on other nodes. The time that k8s waits for a pod to come back online is called `pod eviction timeout` and it's set on the `kubernetes controller manager`. This is 5 mind by default. If the node comes back up after 5 mins, it comes back online as empty and the pods aren't recreated. 

Hence do a quick update and reboot if you cannot be sure if the node is going to be back online in 5 mins.

Way to do it is by draining the nodes `kubectl drain <node-name>`. When you drain a node the pods are gracefully terminated and recreated on another node and the node is marked as un-schedulable. When it comes back online, it is still un-schedulable. You have to un-cordone it `kubectl uncordon <node-name>`. The cordon command simply marks the node unschedulable but does not move the exiting pods to another node, only marks it so that no new pods are scheduled on it `kubectl cordon <node-name>`.

> Drain may need --force or --ignore-daemonsets if the node has any pods running on it that aree not controlled by a replicaset or/ and if there are daemonset pods running on the node respectively.

# Kubernetes version upgrade

ETCD and CoreDNS components are components that are external dependencies used outside of the core components of k8s. The core components are:
1. kube-apiserver --> version x
2. controller-manager --> version x-1
3. kube-scheduler --> version x-1
4. kubelet --> version x-2
5. kube-proxy --> version x-2
6. kubectl --> x+1>x-1

An upgrade can be upgraded component by component. At any time k8s supports only 3 minor versions. Meaning that ver 1.13 supports 1.13, 1.12 and 1.11. The recommended upgrade path is from one minor version to the next and not a big leap to a newer version. Upgrade steps are as follows:

1. Upgrade kubeadm.

```bash
$ apt-get upgrade -y kubeadm=1.13.0
```

2. Upgrade master node. This doesn't have an application downtime but neither the kube-apiserver (hence kubectl) nor the controller managers are active. Since the controller managers aren't active it means that if a replica set goes down, they will not be recreated. 

```bash
$ kubeadm upgrade apply v1.13.0

# if the master node also runs containers, and hence a kubelet process, upgrade the kubelet on the master as follows:

$ apt-get upgrade -y kubelet=1.13.0
$ systemctl restart kubelet
```

3. Update worker nodes. This is basically upgrading the kubelets. Worker update strategies:

    - Strategy 1 - all at once
    - Strategy 2 - Cyclically upgrade nodes
    - Strategy 3 - Add new nodes

```bash
# drain the worker node
$ kubectl drain <node-name>

# upgrade kubeadm and kubelet
$ apt-get upgrade -y kubeadm=1.13.0
$ apt-get upgrade -y kubelet=1.13.0

# run kubeadm to upgrade the node configuration to a new kubelet version
$ kubeadm upgrade node config --kubelet-version 1.13.0

# restart the kubelet service
$ systemctl restart kubelet

# make the node schedulable again. New pods or pods being recreated will come on this node
$ kubectl uncordon <node-name>
```

## Backup and restore

### Resource Configuration

#### Resource Configurations

You must back up resources created using both declarative and imperative methods.

```bash
$ kubectl get all --all-namespaces -o yaml > all-resources.yaml
```

> Vellero can help taking a backup as well.

#### Etcd resource

Etcd has details of all the configurations on the server. Etcd is deployed on the master node. The data directory of the Etcd service has the path for all the configurations in the server and is hence what should be backed up.

`etcd.service --data-dir`

Etcd also has a built in utility for taking a snapshot of the etcd database. 

**Backup**

```bash
$ ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

Example:

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /tmp/snapshot-pre-boot.db 

```

**Restore**

1. Stop the apiserver

```bash
$ service kube-apiserver stop  #   api-server needs to be stopped before doing a restore.
```

2. Restore using the etcd server restore utility

```bash
$ ETCDCTL_API=3 etcd snapshot restore snapshot.db \
--data-dir /var/lib/etcd-from-backup \ # path to the new data directory to be created. This is where the new data will be for this restored database
--initial-cluster master1:https://<master1-ip-address>:2380,master2=https://<master1-ip-address>:2380 \ # master node communication addresses
--initial-cluster-token <target-cluster-token>   #   this is a new cluster token and has nothing to do with the old cluster
```

> When etcd triggers a restore it initialises a new cluster configuration and configures the members of etcd as new members to a new cluster. This is to prevent a new member from accidentally joining an existing cluster. For example when you're setting up a new test cluster with the production cluster configuration, this will ensure that the new members don't accidentally join the production cluster.

3. Pointing the running etcd service to the new restored configuration

Update the etcd.service `--data-dir` and the `--initial-cluster-token` to point to the new configuration. Then restart the etcd service as follows:

```bash
$ systemctl daemon-reload
$ service etcd restart

```

4. Restart the API service

```bash
$ service kube-api-service restart
```

> With all etcdctl commands, you must specify the --endpoint=<etcd-service-address> --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key and the easiest way to do that is to describe the etcd pods before starting all this and saving the value of the ca, server key and server crt in system variables and them use them in the command.

```bash
export servercert=/etc/kubernetes/pki/etcd/server.crt
export cacert=/etc/kubernetes/pki/etcd/ca.crt
export serverkey=/etc/kubernetes/pki/etcd/server.key

# Initial cluster = master=https://172.17.0.20:2380
# Initial advertised peer urls = https://172.17.0.20:2380
# name ingress-space
# data-dir = /var/lib/etcd

ETCDCTL_API=3 etcdctl --cacert=$cacert --key=$serverkey --cert=$servercert --name=master --data-dir /var/lib/etcd-new --initial-cluster=master=https://172.17.0.20:2380 --initial-advertise-peer-urls=https://172.17.0.20:2380 --initial-cluster-token=new-etcd-1 snapshot restore /tmp/snapshot-pre-boot.db

```

5. Now you have to get the etcd pod (or static pod) to point to the new configuration. The following parameters need to be changed:
        - `initial-cluster-token`
        - `data-dir`
        - `volume` - path to the folder where the etcd data is kept

## Cluster HA - Multiple masters

When the master goes down, the pods on the worker keep running and hence the application is available. But if a Pod dies, the ReplicaSet will not restart it because the ReplicaSet controller manager on the master is unavailable. Hence the HA configuration comes from having a HA master control plane.

Masters can be run in Active Active mode, and then have a LB in front to direct requests between them.

API Server - kubectl command talks to the API server. The API server can run in HA across multiple master nodes with a load balancer in front. You may use nginx, ha proxy or anything

Controller Managers - they watch the state of the controllers and take necessary options. Multiple of these can end up duplicating things. Runs in Active Passive

Kube-scheduler - Runs in Active Passive.

Active Passive components work by the leader election process, and the leader-elect must be set to true in the service parameters. The leader-election process tries to get a lock on an endpoint called the kube-controller-manager endpoint. Whichever process first updates the information with this endpoint, gains the lease and becomes the active member. The leader holds the lock for the duration as specified by the leader-elect-lease-duration (default 15s). The active process renews the lease every 10 sec which is the value of the flag --leader-elect-renew-deadline. Both the processes try to become the leader every 2 seconds based on the --leader-elect-retry-period option (2 sec). The scheduler follows the same process and uses the same keys. 

Etcd - 2 topologies

- Stacked Control Plane Nodes Topology - One etcd per master node. Easier to setup and manage. But even if one of the nodes goes down, both the control plane and the etcd member are lost. 
- External ETCD topology - less risky, harder to setup, requires more servers. 

> API Server is the only component that talks to the ETCD server and hence that can be configured on the APIServer Configuration. 

ETCD is a distributed server and hence can be reached from any of the instances. Read Write can be done on either of the ETCD nodes. API server ETCD config takes the 2 server addresses.


## ETCD HA Double click

ETCD is a distributed reliable key-value store that is Simple, Secure and Fast. 

ETCD reads and writes consistently on distributed nodes. ETCD does not process the writes on each node. Internally the nodes elect a leader and the other nodes become the followers. If the writes come in through  the leader, it writes it and shares it with the others. If the write goes to one of the followers, the follower internally forwards it to the leader, which writes it and then passes it to all the followers. Thus the write is only considered complete when the leader gets a consensus from all the nodes. ETCD implements distributed consensus using the RAFT (Random ) protocol. 

The process:

- 3 node etcd Cluster with no leader elected
- RAFT uses random timers for initiating requests.
- First all 3 nodes generate a random timer, when the first node finishes the timer, it sends a request to the other nodes for them to give consensus on if the node should become the leader.
- The nodes respond, and on receiving their vote, the node becomes the leader.
- The leader sends out messages at regular intervals to the other master nodes telling them that it is the leader.

If the other nodes don't receive a message for some time, the nodes initiate the leader election process 

A write to ETCD is considered to be completed only when it is replicated across a majority of the multiple etcd nodes. Example: in a 3 node etcd cluster, if the write succeeds on 2 nodes, it is considered to be a success. This majority is called the quorum. The quorum is (total number of etcd nodes devided by 2) + 1. If the value is a decimal value, take the integer value as the quorum value. Quorum of 1 is 1, 2 is 2, 3 is 2, 4 is 3, 5 is 3

> It is hence recommended to have at least 3 nodes in an ETCD cluster so there can always be quorum.

### Fault tolerance

The total number of ETCD nodes minus the number of nodes required for a quorum is called the fault tolerance. This is the number of etcd nodes that a cluster can afford to lose while still being able to achieve quorum.

> Because the fault tolerance of a 3 or 4 (tolerance - 1) and a 5 or 6 (tolerance 2) nodes cluster is the same, it is recommended to have an odd number of etcd nodes in an HA cluster. 

Obviously, if you ran multiple etcd instances on the same k8s node, it would be pointless because losing the node wouldn't leave you with any HA, hence you'd setup an ETCD HA cluster on separate nodes. And because the ideal setting is an odd number of nodes for ETCD, the number of nodes required to run an HA etcd cluster is 3. 