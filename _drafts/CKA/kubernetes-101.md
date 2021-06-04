---
layout: post
categories: [Writeups, Kubernetes]
comments: true
excerpt: A primer guide with commands and syntax for kubernetes
title: Kubernetes 101
tags: [kubernetes, cloud native]
description: A primer guide with commands and syntax for kubernetes
---

## Kubernetes Components

### Master Node Components

1. etcd - key value store that holds all the data about the cluster, the pods, containers, versions, deployment times etc. Default port 2379
2. kube-scheduler - the component that manages deployment of containers on a node, identifies the right node as per the policies etc.
3. api-server - This is the API server that provides the internal and external interface for clients to talk to Kubernetes. This component is called everytime you fire a command from the cli, the dashboard, and is even used by internal components for sharing and receiving the latest configurations of the cluster.
4. Controllers - monitors the state of resources and works to bring them to the desired state
   - Node Controller - Node status - monitors nodes every 5 seconds; waits for 40 seconds before markingit unreachable. 5 mins wait after that for machine to come back. Then it'll remove pods from node
   - Replication Controller 
   - Cloud Controllers


### Worker Node Components

1. kubelet - This is a process that runs on every worker node of the cluster, with the singlular capability of running pods and setting up the basic necessities such as the storage, networking and compute runtime required by the containers.
2. kube-proxy - this is a proxy server that runs on every node and does simple or round robin TCP, UDP or SCTP forwarding between the incoming traffic and the backend set running on the node. 

## Resources


### Pods

Pod definition default yaml vars:

```yaml
apiVersion: v1
type: Pod
metadata:
  name: <pod_name>
  label:
    app: something
spec:
  containers:
  - name: <somename_for_container>
    image: karankapoor/container_name
    ports:
    - containerPort: 8080

```



#### Useful Commands

- Getting the details of a pod as a yaml
  `$ kubectl get po <pod_name> -o yaml`
- Getting more details about a deployed pod
  `$ kubectl get po <pod_name> -o wide`
- Watch the get pods command for changes to pods
  `$ kubectl get po <pod_name> -w`
- Imperatively way - refer to kubernetes CLI resource creation guide
- Get pods by specific labels
  `$ kubectl get po --selector <key>=<value> `  # instead of usign --selector, you can also use -l

- Edit a pod
  `$ kubectl edit pod <podname>`     # you cannot however edit some of the properites for a pod at runtime such as the resource utilization, env vars, service accounts etc.
  * export the pod definition to a yaml file, make changes and then delete the pod and recreate using the modified yaml file. A better option is to put the pod into a deployment and editing the deployment file since the pod spec is a child of the deployment. 


### ReplicaSet

```yaml
apiVersion: apps/v1
type: ReplicaSet
metadata:
  name: <Replicaset_name>
  label:
    app: something
spec:
  replicas: <number_of_base_replicas>
  selector: 
    matchLabels:
      app: something
  template:
  # Copy the pod definition here
    metadata:
      name: <pod_name>
      label:
        app: something
    spec:
      containers:
      - name: <somename_for_container>
        image: karankapoor/container_name
```


the labels under the matchlabels section must match the pods labels and not the rs labels because these are used by the rs to determine which pods to control.

#### ReplicaSet vs. ReplicationController

> One of the major differences b/w replication controller and the replicaSet is the selector label. Selector labels help the replicaset control pods that weren't created using the template inside the RS definition. The Selector label is an optional component in the RC. 
rc are available in apiVerision v1, while rs is available in apps/v1. Here's a difference from a selector perspective.

```yaml
# replicationController - this is the old contructor that only has an equal to operand such that the key and value should match exactly
apiVersion: v1
spec:
  selector:
    app: frontend  #  the label on the pod must be an exact match

# replicaSet - this is the newer constructor, now replaced by a deployement. This allows for using operators in the selection of pods to be controlled by the set
apiVersion: apps/v1
spec:
  selector:
    matchLabels:
      app: frontend
    matchExpressions:
    - key: app  #   The value of the app label could be any pf frontend or blue
      operator: In
      values:
        - frontend
        - blue
    - key: tier  # The value of the tier label on the pod should neither be production nor UAT
      operator: NotIn
      values:
      - production
      - UAT 
```


### Deployment

A deployment is a superset to a replicaSet. While a replicaSet creates a unit that allows for recreation of dead pods, it leaves no way for you to roll back the state to an old state if the latest set of pods you've deployed needs to be rolled back. Rolling deployment was a key capability of the replication controller that is not available in replica sets. Deployments are now the standard way of creating pods. A deployment automatically creates the replication set, a deployment, and the pods. 

Deployment also help with rollout and rollbacks of pods. A Deployment internally creates a replicaSet for the deployment of the pods. When a new rollout is created, the deployment creates a new ReplicaSet and replaces the old pods with new ones as per the new replicaSet based out on the rolloutStrategy. When you roll a release back, the deployment uses the rollout strategy to create pods based on the old replicaSet while removing the pods from the current one. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  label:
    app: backend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      name: my-pod
      labels:
        app: backend
    spec:
      containers:
      - name: backend-container
        image: my-image
        ports:
        - containerPort: 80   # tells the container which port to expose

```

#### Useful commands

- `kubectl run <deployment-name> --image=<image-name> - l "<key:>:<value>,<key>:<value>"` - creates a deployment
- `kubectl rollout status deployment <deployment-name>` - get status of the deployment process
- `kubectl set image deployment <deployment-name> <container-name>=<new-image> --record ` - updates the image of the container inside the deployment while recording the command in the rollout deployment history
- `kubectl rollout history deployment <deployment-name>` - shows the history of a deployment 
- `kubectl scale deploy <deployment-name>


### Namespaces

Namespace service Address: <namespace>.svc.cluster.local
kube-system: namespace for k8s services
kube-public: services to be accessed by all namespaces
default: a default namespace


Switch your current context to a different namespace

```bash
$ kubectl config set-context $(kubectl config current-context) --namespace=<namespace_name>
```

### Services

**Nodeport**: makes container port available on the node's physical port

**ClusterIp**: Service creates a virtual IP inside the cluster to enable inter container communication.

**LoadBalancer**: use a cloud provider's nodeport

Services is how kubernetes allows you to expose services both within and outside the cluster. Generally speaking a Service refers to a ClusterIP service.

- `ClusterIP`: The ClusterIp basically assigns a virtual IP to your resource from a pool of IP's that your CNI has provided to each node to provide to Pods. Note that no matter what kind of service you create, a clusterIP service is always created, and the service type you create binds to that internally.

- `NodePort`: makes container port available on the node's physical port of the worker nodes. By default, the same port will be selected and exposed on all the worker nodes and the service will then be reachable on that port on every worker node IP address. The nodeport service binds to the clusterIP of the service for routing, hence every time you create a nodeport service, an ensuing clusterIP service is also created.

- `LoadBalancer`: creates a cloud provider based load balancer which has it's own IP address. This LB could be exposed to the internet or not as per your choice. The LoadBalancer service balances the load between multiple NodePort services by distributing traffic on the port exposed by the NodePort Service. Point to note here is that if you have multiple microservices and you plan on exposing all of them, using a LoadBalancer type service for each of the services can tunr out pointlessly expensive. Essentially, a Load balancer should be able to handle multiple backend services to route to depending on the requested port, hostname or application route. This functionaltiy is not built into kubernetes and one can choose from amongst various tools to do this including nginx, istio, weave works, calico etc. by creating something known as an ingress controller in the cluster and an ingress resource for every individual service that needs to be exposed.

- `Ingress`: An ingress service can typically provide Layer 3, Layer 4 and Layer 7 based routing for applications on the same LoadBalancer infrastructure thereby utilizing LoadBalancers more efficiently. The ingress controller essentially applies the traffic redirection rules as a yaml resource once the traffic enters your cluster through the Load Balancer set up on the cloud provider.

Most services (except ClusterIP and Ingress) will have the following fields: 

  - `TargetPort`: the port on which the pod is exposing what's running inside it. So for example if the service inside the container is an httpd server hosting a page on the default port 80, the TargetPort would be port 80. Basically target port is the port that has to be targeted by the service which is basically the port being exposed by the pod that is going to be fronted by the service.

  - `Port`: this is the service port. This is the port on which the service will be available inside the cluster. For example is my port is 9090 and my service name is test deployed in the default namespace, it can be reached at test.default.svc.cluster-local:9090 from inside the cluster.

  - `NodePort`: A valid port in the 30000-32767 port range on the host machine.


By default, k8s uses a "Random" Algorithm. It makes the same port on all nodes available as the outbound port for the service.

```yaml
apiVersion: v1
kind: Service
metadata:
    name:
    label:
spec:
    type: NodePort
    ports:
    - targetPort: 80  # taken as the same value as the port if not specified
      port: 80  # mandatory
      nodePort: 30008  # will be selected from the valid range if value is not provided
    selector: # selector labels from the pod definition
      app: something
      type: something
```


**ClusterIp**: the right method for establishing internal communication between pods. This basically gives a single interface/ internal IP address for other pods to access the service. Uses " Random" algorithm.


```yaml
apiVersion: v1
kind: Service
metadata:
    name:
    label:
spec:
    type: ClusterIP  # every service even when the type is not defined, defaults to clusterIP
    ports:
    - targetPort: 80 # taken as the same value as the port if not specified
      port: 80  # mandatory
      nodePort: 30008  # will be selected from the valid range if value is not provided
    selector: # selector labels from the pod definition
      app: something
      type: something
```

> Important: to create a dummy template yaml file than can be used to fill up with more specific details before actually using out the file, use the following:

```bash
$ kubectl expose deployment <deployment-name> --type=<service-type> --port=<service-cluster-internal-port>  --dry-run -o yaml > service.yaml   # this creates a file called service.yaml which has the definition for the service that you want to create.
```


#### Expose pod as a service

```bash
$ kubectl expose pod <podname> --port=<portnumber> --name=<service-name> --type=NodePort  //automatically uses the pod's selectors for the service.
```

#### Create a deployment 

```bash
$ kubectl create deployment --image=<imagename> <deployment-name>
```

> Note that this command doesn't allows you to define the number of replicas. You will either need to create the file or use the scale command

```bash
$ kubectl scale deployment/<deployment_name> --replicas=4    //note that the scale command doesn't update the underlying yaml with the latest number of replicas in the deployment.
```


### Kubernetes Scheduler

Scheduler decides which pod should go on which node. It does so by setting a property called nodeName in the pod definition file automatically if it hasn't already been set. Without the scheduler, the pod stays in a pending state and is not deployed to any particular node.

You can manually set the `nodeName` field in the Pod `spec`. 

> You can only specify the node name property at creation time. It cannot be modified later. 

To be able to change the nodeName spec for a pod, you should configure a binding object that binds the pod to a node. This is essentially the same way how the scheduler does it.

So if you wanted to specify the nodeName in the Pod spec itself, it would look like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: <pod-name>
    label:
        app: frontend
spec:
    containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 8080
    nodeName: node02
```

Conversely if you'd want to create a binding to do the pod to node allocation, a *representative* Binding yaml would look like:

```yaml
apiVersion: v1
kind: Binding
metadata:
    name: <my_binding_name>
target:
    apiVersion: v1
    kind: Node
    name: node02
```

> The above is *only for representation purposes*, in order to actually do this, you have to send a REST API call with the input as json formatted string.


### Labels and Selectors

Labels are the key value labels that you put in your cluster. Selectors are what are used to select specific components basis their types and labels.

> Annotations are another section that can be given in a resource's metadata definition. This can be used to specify things like the author, version or something else that you may want to specify. Annotations are also sometimes used by custom resources to enable/ disable and configure certain features.

### Taint and Tolerations

Taint is marking a node such that the scheduler doesn't schedule the pod on the node. There could however be pods that can be tolerant to the taint and can still be placed on a particular node.

Use Case: Imagine you've got 2 services, A and B that need to be deployed on the cluster. Node 1 is a dedicated resource in your 3 node cluster for Service A. In order to achieve this isolation, here's what you'd do:
- Taint node 1 so no pods can be placed on that node.
- To enable pod A to be able to be deployed to this node, add a toleration to Service A pod that can tolerate the taint. 

> Taints are set on nodes, tolerations are set on pods.

> Taints and tolerations do not guarantee that a pod with a certain toleration will not be placed on any other un-tainted node. Meaning that a node tainted red will only accept pods which have a toleration defined for red, but it doesn't imply that the pod with a toleration for red will not be placed on a node that doesn't have any taint.

In order to do this, create a taint on a node:

```bash
$ kubectl taint node <node_name> <key>=<value>:<taint_effect>

# <key>=<value> is how you create a taint. For any pod to be deployed on this node, it must have this <key>=<value> in its tolerations.

```

3 types of taint_effects

- `NoSchedule` - no new pods will be scheduled on the node; existing pods will remain.
- `PreferNoSchedule` - it will attempt to not schedule any pods on the node, but not guaranteed. Existing pods will remain
- `NoExecute` - No new pods will be scheduled; existing pods if they don't have a toleration towards the taint, will be gracefully evicted.

Example command:

```bash
$ kubectl taint node node1 app=blue:NoSchedule   # Note the absence of the space and the taints key=value pair
```

To add tolerations to a pod, add tolerations to the spec section in the pods definition as follows:

> This is how one would create a toleration for the node1 as per the example command above. All values MUST BE in double quotes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod_name>
  label:
    app: something
spec:
  containers:
  - name: <container_name>
    image: <image_name>
    tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
```

NoExecute: As soon as the taint with NoExecute is defined on a node, it will automatically kill the pod that doesn't have the toleration for that specific taint.

> Point to remember: Taints and tolerations do not limit the pods from going to a specific node, instead limit the kind of pods that a node can accept. This means that if Node1 has a taint and only Service A has a toleration for that taint, the taints and tolerations stop pods other than Pod A from being scheduled on Node1 but does not stop Pod A from being scheduled on any other node. This can be done using Node Affinity.

> By default in a k8s cluster, there is a taint set on the master node that prevents pods from being scheduled on the master node. See it using the command `kubectl describe node master | grep Taints`

To remove a taint from a node, get the value of taint from the node description and run the following command:
`$ kubectl taint node <node-name> <taint-value-from-node-description>-    # Note the hyphen at the end of the command`


### Node Selectors

You may have different hardware boxes as a part of the same cluster that you may want to dedicate to different pods etc. The simple way to do it is to define the `nodeSelector` in the pod description with the key value pair that you also mark on the specific node where you want these pods to go. 

Labelling a node:

```bash
$ kubectl label nodes <node-name> <key>=<value>
```

In the pod definition yaml, you can do the following: 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod_name>
  label:
    app: something
spec:
  containers:
  - name: <container_name>
    image: <image_name>
  nodeSelector:
    <key>: <value>           
```

> This can help you put specific pods on specific nodes, but what if you want a slightly different scenario where you want a pod to be deployed to a node with the labels size=small or size=medium? or a limitation like any node except size=small? - This can be done using Node affinity.


### Node Affinity

You can provide advanced/ complex selector mechanisms for scheduling pods on nodes using node affinity. This is where you can use operators as well to for example specify a set of nodes where the values for the key may be x, y or z.

> Node affinity and anti-affinity can also be used to create hard or soft affinity on the basis of pods such as a hard anti-affinity for pods A and pods B being placed on the same node together.

- Hard Affinity: requiredDuringSchedulingIgnoreDuringExecution - these rules MUST be satisfied for a pod to be scheduled to a node.
- Soft Affinity: preferredDuringSchedulingIgnoreDuringExecution - the scheduler must try and meet these rules but continue to schedule the pod even if they're not met.

**Roadmap Affinity Types**

- RequiredDuringSchedulingRequiredDuringExecution - this will evict the pods from any of the nodes where the conditions aren't met.

> IgnoreDuringExecution means that if the labels on a node change at runtime, the pods already running on that node that do not have their rules met, will continue to keep running and not shut down.

Supported Operators:
- In
- NotIn
- Exists
- DoesnotExist
- Gt
- Lt

> One can use NotIn and DoesNotExist along with Taints to achieve node anti-affinity.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod_name>
  label:
    app: something
spec:
  containers:
  - name: <container_name>
    image: <image_name>
  affinity:
      nodeAffinity:
          requiredDuringSchedulingIgnoreDuringExecution:
              nodeSelectorTerms: # This will check if the node matches ANY of the matchExpressions. It does not need all the match expressions below it to be met.
              - matchExpressions:  # This will ensure that the pod is scheduled on the node that matches ALL the key value pair conditions in the matchExpressions
                  - key: <somekey>
                    operator: In # In case you use Exists, you do not neceseily need to give a value because it'll check if the key itself exists.
                    values: 
                    - somevalue1
                    - soevalue2           
```
> NOTE: yaml doesn't accept tabs and typing out this yaml can be quite difficult typographically. Try and type this on a yaml editor if required.


```yaml
# Representation for setting node affinity by required during scheduling rule
affinity:
    nodeAffinity:
        requiredDuringSchedulingIgnoreDuringExecution:
            nodeSelectorTerms: # Non Array type
            - matchExpressions:     # Array of match expressions, ALL match expressions in the array must be satisfied
                - key: <somekey>
                  operator: <some_operator>
                  values:
                  - <value1>
                  - <value2>
```

Weight is needed when setting up a preferredDuringScheduling rule because the scheduler will try and schedule the pod with the highest score:

```yaml
# Representation for setting the node affinity by preferrence rule
affinity:
    nodeAffinity:  
        preferredDuringSchedulingIgnoreDuringExecution:
        - weight: 100     # Array starts from here, not from matchExpressions. Weight is calculated for every individual node and the highest scoring node gets the pod.
          preference:
            matchExpressions:
            - key: <somekey>
              operator: <some_operator>
              values:
              - <value1>
              - <value2>
```

> Node anti-affinity doesn't have a separate clause for itself. You control it using the operators in the 

> Node affinity is defined on the pod spec, meaning that that pod will be placed on a node with a matching label, it doesn't however restrict other pods from being scheduled on that node. 

### Pod Affinity

We may want to put for example the redis pod on the same node where the frontend pods are. Hence, similar to node affinity, one can define pod affinity and anti affinity to place pods in proximity/ away from each other. 

```yaml
affinity:
    podAffinity:
        requiredDuringSchedulingIgnoreDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: <some-key>
              operator: <some-operator>
              value: <some-value>
          topologyKey: failure-domain.beta.kubernetes.io/zone
```

```yaml
affinity:
    podAntiAffinity:
        preferredDuringSchedulingIgnoreDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
                matchExpressions:
                - key: <some-key>
                  operator: <some-operator>
                  value: <some-value>
            topologyKey: zone/ hostname/ region
```

> GOTCHA: Let's assume a scenario where pods S1 and S2 need to be placed together, hence you're looking to setup pod affinity for the same. In such a scenario where the podAffinity is set to a particular label for say a pod S1 and the affinity rule for S1 is written in pod S2's podSpec, the first time you try to create a pod for S2 and if there are no running S1 pods, S2 will also not get scheduled if the affinity rule is set to requiredDuringScheduling. For this corner case, see https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md#affinity 

`topologyKey`: these are pre-populated labels allocated to nodes and persistentVolumes (more on that later) and you can use these labels to define nodeAffinity and pod affinity. For more information and a detailed description of these labels, see : https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesioregion 


### Dedicating nodes for specific pods

To dedicate nodes for specific pods, we must:
Step 1. - use taints and toleration to paint the individual nodes
Step 2. - use node affinity to get specific pods to specific nodes



### Resource requirements and Limits

> If during scheduling, the scheduler finds that no capacity is available on any of the nodes, it delays the scheduling of pod on any of the nodes and the pod shows in pending state.

> By defualt k8s assumes that every pod requires 0.5 CPU's and 256 MB RAM. - this is the minimum requested by every pod. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <somename>
spec:
  containers:
  - name: <container_name>
    image: <image_name>
    ports:
    - containerPort: 8080
    resources:      # resource configurations are defined per container.
        requests:       # container in the pod is initiated with this set of resources. Basically the minimum
          memory: "1Gi"     # Ki (Kibibyte) = 1024 bytes while K is 1000 bytes
          cpu: 1
        limits:     # container in the pod can go upto this maximum limit on the node.
          memory: "2Gi"
          cpu: 2 

```

Docker containers can typically take up all the resources on a node. K8s however sets a 1 vcpu limit. 
If a pod is throttled to stay in the cpu limit however k8s allows the pod to go over the memory limit. If the pod stays beyond the memory limit for longer durations, it is terminated.


### Daemon Set

Daemon sets allow you to run a particular pod on every node of the cluster. Services such as kube proxy, some per node networking agent, a monitoring and logging agen tetc are the perfect examples of Daemon sets. The creation and deletion of a doaemon set is linked with the creation and deletion of the node. It will be created when a new node is attached and deleted from that node when the node is removed.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: <some-name>
spec:
  selector:
    matchLabels:
        app: monitoring-agent
  template:
    metadata:
        name: <somename>
        labels:
          app: monitoring-agent
    spec:
      containers:
      - name: <container-name>
        image: <monitoring-agent-image-name>
```

Daemon sets internally use the nodeAffinity rules to ensure one Daemon set per node. 

### Static Pods

If one looks at the individual worker nodes, every worker has a kubelet process running on it that knows how to run pods on the machine. In order for it to run a pod however, it must receive the yaml file from the api-server running on a master node. We can configure a path on the worker machine from where the kubelet can pick up yaml files and create pods wihtout needing the api-server. The kubelet periodically checks the folder for changes and runs any new resources. If the pod fails, the kubelet can recreate the pod on the node. If you make changes, the kubelet applies the changes. If you delete a file from the directory the pod will be removed as well. Such pods are called static pods.

> kubelet can only run a pod this way, replica sets, deployments, services etc can not be run this way. 

1. Configuring using a config file:
The kubelet service has a path for the configuration file that it is using. `--config` in the kubelet.service. The config yaml file that this flag points to should have an entry called the `staticPodPath` for the folder from where the kubelet should pick up files for creating static pods. Cluster configured using the kubeadm tool use this method. 
To check static pod path:
```bash
$ ps -aux| grep kubelet
# read the location of the --config flag
$ cat <file_location>/<filename> | grep staticPodPath
```


2. Configure using the kubelet service:
The kubelet.service file can have a variable called the `--pod-manifest-path` which defines the folder path for picking up yaml files for static pods

> In the worker node, we don't have the kube-api server, the pods run using the kubelet will still be seen using the `docker ps` command. Kubectl commands will not be available here for the management.


> The api server on the master talks to the api exposed by the kubelet. Hence one can create static pods using the API for the kubelet as well. 

Static pods create using the kubelet directly show up in `kubectl get pods` as well but are in read only mode and appended with the name of the node. Management will have to be done from the individual files on the node itself.


> Use case: this can be used to deploy pods on a node such that they're not dependent on the kubernetes control plane. We can use it to deploy the various control plane components as well. That's how kubeadm does the setup. You install kubelet, create pod defiition file for docker and then put into the static pods folder. 

#### Static Pods(SP) vs Daemon Sets(DS):

SP: Created by Kubelet
DS: Created by Kube API server using the DaemonSet Controller

SP: Used to deploy control plane components as static pods.
DS: Deploy Monitoring, Alerting etc. per node agents

Both are ignored by kube-scheduler. the kube scheduler has no effect on these resources.


### Kube schedulers

We can use a custom scheduler that can be used to schedule pods as per our own custom logic. We can give schedulerName in the podSpec to specify which scheduler we want to run for the specific pod. 

If the scheduler is misconfigured, the pod will remain in a pending stage and will not be scheduled onto a node. To find the scheduler, on the individual kubernetes worker node, look at the kubelet service details by looking at the kubelet service startup command, and look for the --config flag for the location of the kubelet configuration file. Here look for the staticPodPath. The kubernetes scheduler yaml file should be here. To create your own scheduler, make a copy of this file, and make changes to it.


1. Get the kubernetes config file path (if you're logged into the master. If not, you might need to ssh into the master node)
```bash 
$ ps -aux | grep kubelet | grep config
# Example Response
root      3602  2.7  2.2 764928 91184 ?        Ssl  08:31   0:07 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf--config=/var/lib/kubelet/config.yaml --cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1
```

3. Open the file at the --config path and get the staticPodPath 

```bash
$ cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Example response
staticPodPath: /etc/kubernetes/manifests
```

4. Copy the kubernetes scheduler yaml from the staticPodPath location

```bash
$ cp /etc/kubernetes/manifests/kube-scheduler.yaml my-scheduler.yaml
```

5. Edit the my-scheduler.yaml spec

```yaml
spec:
  containers:
  - containers:
    - kube-scheduler
    - --scheduler-name=my-scheduler  # Additional Parameter
    - --port=10282  # Additional Parameter
    - --leader-elect=false  # Set as false in case you have multiple schedulers but a single master. Set to true if you have multiple masters
    - --lock-object-name=my-scheduler   # Set in case you have multiple schedulers and multiple masters
    livenessProbe:
      failureThreshold: 8
      httpGet:
        port: 10282  # Updated port number
```

6. Deploy the new scheduler and check running status

```bash
$ kubectl create -f my-scheduler.yaml
$ kubectl get po -n kube-system | grep my-scheduler
```

#### Using the scheduler in a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  schedulerName: my-scheduler
  containers:
  - name: nginx
    image: nginx
```


### Logging and monitoring

Heapster was the original solution for running metrics on a k8s cluster but that has now been depricated and a slimmed down version called Metrics Server is now used. 
The kubelet on every node is responsible for reporting the metrics to the kube api server intermittently. It internally uses something called the cAdvisor (container advisor) to get this data. Note that metrics server is a completely in-memory solution and hence cannot give historic reporting.

To deploy the metrics server, clone the git repo for metrics server and run the yaml files to deploy the metrics server to the cluster. After its deployed, you need to give the pods some time to collect the data. Then run 

- `kubectl top node` to show node level metrics

- `kubectl top pod` to show pod level consumption metrics


### Deployment and rollouts

When yoy create a new deployment, a rollout is created. When you do a rollout then, a new deployment version is created every time for a new revision. 

```bash
$ kubectl rollout status deploy <deployment_name>   # tracks the rollout of a deployment

$ kubectl rollout history deploy <deployment_name>    # something like a git history of all the rollouts on the deployment

```

#### Deployment strategies

1. Recreate - Delete all pods, then create new ones. This is obviously not the default strategy as the service who's pods have been deleted will be unavailable till the new pods come up

2. Rolling update - Maintains a minimum a max number of pods at any given time while it kills pods systmatically and replaces them with new ones. This is the default deployment strategy.

Apply changes to a deployment by:
1. Changing the image in the deployment yaml descriptor file and then using `kubectl apply` to apply it
2. Using the `kubectl set image deploy <deployment_name> <container_name>=<new_image_name>`. This however will not make the changes to the descriptor file and hence can lead to a synchronization mismatch between the actual deployed and the file.

The rolling update strategy creates a new Replica Set in the deployment with the new deployment parameters. It then starts scaling down the number of replicas in the old replica set to a 0 while incresing the number of pods in the new replica set. 

> But what if theres a mistake in the new deploynment. Rollback will do the opposite of the rolling update by scaling up the pods in the old replica set, replacing the pods from the new replica set.

#### Rollback

```bash
$ kubectl rollout undo deploy <deployment_name>
```


### Correlating docker and k8s and configuration

#### Docker background

There are 3 basic commands in the dockerfile setup that specifcy what runs in the container.

1. RUN - this is used to run commands that are used for setting up the container, such as downloading dependencies and packages etc. This creates a new layer on your base image.
2. ENTRYPOINT - specifies the command that is run when a container starts. For example ["sleep"]. This tells the container to run the sleep command as soon as it starts. The command however needs a parameter for it to work (the duration to sleep for in this case.)
3. CMD - this takes an array format for the command that needs to be run when the container starts. It is best used in tandem with the ENTRYPOINT. When used alone, it can get the container to sleep for 5 seconds as so ["sleep","5"]. When used in tandem with ENTRYPOINT, the arguments would be simply ["5"]. 

Then why use entrypoint at all?

Without entrypoint this is what the docker command would look like. CMD ["SLEEP","5"]

```bash
$ docker run sleeping_ubuntu    # Start the container and sleep for 5 seconds
# Now we want to have it sleep for 10 seconds instead
$ docker run sleeping_ubuntu start 10   # Notice how you need to pass the whole command as an argument to ovverride it. 
```

With entrypoint 

ENTRYPOINT ["sleep"]
CMD ["5"]

```bash
$ docker run sleeping_ubuntu    # start container and get it to sleep for 5 seconds
# overriding to sleep for 10 seconds
$ docker run sleeping_ubuntu 10   # overrides the setting in the CMD property

# Note: the entrypoint can also be overridden by passing in the CLI property --entrypoint
```

#### Playing this in tandem with kubernetes

In the kubernetes pod definition file, for each container, 

1. command:["sleep"] - this replaces/ overrides the entrypoint docker command
2. args:["5"] - overrides the CMD fromt he dockerfile configuration


#### Passing Environment variables

When running docker containers, the way to pass environment variables would be as follows

```bash
$ docker run -e <variable_key>=<variable_value> <container_name>
```

To do the same thing in k8s, one can do this in the env array in a container definition in a pod definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  label:
    app: myapp
spec:
  template:
    containers:
    -  name: my-container
       image: <image_name>
       env:
       - name: <key_name>
         value: <value>
```

Additionally, the value can come from a configMap or a secret in k8s. Instead of `value`, use the key `valueFrom` instead like so:

```yaml
# to get value from a config map
env:
- name: <key_name>
  valueFrom:
    configMapKeyRef: <refrence to configmap key>

# to get value from a secret
env:
- name: <key_name>
  valueFrom:
    secretKeyRef: <refrence to configmap key>
```


### Passing parameters into Containers

#### Using ConfigMaps

Config maps are simple key value pair for passing data to kubernetes yaml files. these can be created in 2 ways:

1. Imperative

```bash
# from the CLI
$ kubectl create configMap <config-name> --from-literal=<key-name>=<value> --from-literal=<key-name>=<value>

# using a properties file
$ kubectl create configMap <config-name> --from-file=configFile.properties
```

Properties file format is as:

```json
APP_NAME: my-app
color: green
```

2. Declerative

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name:
data:
  <key-name>: value
  <key-name>: value
  
```

Then just use the `kubectl create -f` to create the configmap.

To use the configMap in a pod, add it to the pod spec as so

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  label:
    app: my-app
spec:
  containers:
  - name: my-container
    image: some-image
  envFrom:
  - configMapRef:
      name: <configmap-name>
```

The above will inject all the key value pairs in the configMap as environment variables into the container. 

There are 2 more ways of injecting variables.

1. Pass a particular varibale instead of an entire config map

Instead of `envFrom`, use `env`

```yaml
env:
  - name: <the to be environment varibale name>
    valueFrom: 
      configMapKeyRef:
        name: <configMap name>
        key: <which key's value from the configMap to be mapped to the variable>

```

2. Mounting a configmap as a volume

```yaml
volumes:
- name: <new-volume-name>
  configMap:
    name: <config map name>
```

#### Using Secrets

1. Imperative Creation:

```bash
$ kubectl create secret generic <secret-name> --from-literal=<key-name>=value --from-literal=<key-name>=<value>
```

or pass `--from-file` and give the path to a properties file container all the properties

2. Declerative Creation:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
data: 
  hostname: <data>
  username: <data>
  password: <data>
```

> Note that encoding the secret values when using the Imperative decleration using the CLI is not required. Kubernetes does that automatically.

In order for the <data> to be human unreadable, it needs to be converted to base64 before inserting into the secret data. This can be done as follows:

```bash
$ echo -n '<data>' | base64   # this works on a linux/ unix machine
# For example 

$ echo -n 'Welcome1' | base64
V2VsY29tZTE=
```

To get and describe secrets use the following:

```bash
$ kubectl get secrets
$ kubectl describe secret <secret-name>
```

The describe command will give the secret value in an encoded format. To decode the value, use the following command

```bash
$ echo -n '<encoded-value>' | base64 --decode
```

In order to inject the secret into a container, do the following:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  - containers:
    - name: my-container
      image: image-name
      envFrom:
        - secretRef:
            name: my-secret
# this will inject the entire secret into the container in the pod
```

Other wats to do it are as follows:

1. As a single environment variable

```yaml
env:
- name: entry-name
  valueFrom:
    secretKeyRef:
    - name: my-secret
      key: key-name
```

2. As a volume

```yaml
volumes:
- name: volume-name
  secret:
    secretName: my-secret
```

When injecting the secrets as a volume into a container, each key is created as a new file inside the container with the data.key as the filename and the data.key.value as the value inside the file. Hence doing a `cat /opt/valume-name/<key-name>` shows you the value of the secret.

> When injecting secrets as a volume, recreating the pod is not necessary. The time taken for the updated value to reflect in the contnainers inside the pod is a factor of the kube-api accepting the value + the sync delay between the api-server and the kubelet. In case of environment variables, 

Since it is this easy to decode a secret, it is obviously inherently not that secure. It hence makes sense to:
1. Not check in your secrets into source code.
2. Enable encryption at rest so they are encrypted even when kept in etcd.

A few important points about secrets though:
1. A secret is only sent to a node if the pod on that node requires it
2. The kubelet stores the password in tempfs and not the file system
3. Once the pod that depends on the secret is deleted, the kubelet deletes the local copy of the secret as well.

> A better way to handling secrets in k8s hence are using other solutions such as Helm secrets or HashiCorp Vault.


### Multi Container Pod Patters

There are 3 multi container pod patters:
1. Sidecar
2. Adapter
3. Ambassador

### Init Containers

Consider the case where you have a container that needs to wait for something to happen before it is initiated. Examples of this would be such as:
- Loading dependencies from another repository
- Ensuring some other service in some other pod is ready
etc. 

There could be multiple such dependencies that need to execute before the actual container execution begins. 

By default kubernetes sees all the containers running inside the pod and continues to kill and restart the pod till all the containers inside it are running succesfully. But in the case described above, the containers would be short lived. Such containers that must run with the actual application container but are somewhat like dependencies to the primary container can be defined in the pod definition in the InitContainers section of the pod definition. It's an array parallel to and similar the contintainers list in the definition. These init containers are executed sequentially one at a time and must finish in order for the primary container(s) to begin executing. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  - containers:
    - name: my-container
      image: image-name
  - initContainers:
    - name: myapp-init
      image: busybox
      command: ['sh','-c','git clone <repo containing dependencies>; done;']
```

