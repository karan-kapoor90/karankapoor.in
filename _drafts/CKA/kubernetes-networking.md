---
layout: post
categories: [Writeups, Kubernetes]
comments: true
excerpt: A primer guide for Networking in Kubernetes
title: Kubernetes Networking
tags: [kubernetes, cloud native, Networking, CNI]
description: A primer guide for Networking in Kubernetes
---


## Networking Basics

To see network interfaces on a host, use the command `ip link`.

Once you have a link that connects to the same switch, say 192.168.10.0, you can assign IP addressed to hosts on that network using the command `ip addr add 192.168.10.10/24 dev eth0`. 

To connect different networks together (assume switch1 @ 192.168.1.0 and switch2 @ 192.168.2.0), you need to use a Router. The router is an intelligent device. It assigns an IP to both the networks (say 192.168.1.1 and 192.168.2.1 respectively)

On a linux machine, run the `route` command to see the network configuration of your machine. To enable traffic from network 1 to network 2, run the command `ip route add 192.168.2.0/24 via 192.168.1.1` where 192.168.1.1 is the gateway for network 1. This needs to be done on every machine in network1.  

To get a network connected to the internet, connect the router to the internet and set a configuration to push all traffic for an un-recognised IP address (such as the address to google.com etc.) to the gateway which is the router. Do this using `ip route add default via 192.168.2.1`, this will add an entry to the machine's routing table.

To make an entry into a machine's etc/hosts file `cat >> /etc/hosts`. To get a machine's hostname, do `hostname`.

Every server has a file called `/etc/resolv.conf` in which you give the DNS servers name against the keyword `nameserver` and the IP address for the DNS server. By default a server will look in the hosts file before looking at the DNS server. This priority can be changed by making a change in the `/etc/nsswitch.conf`. A free nameserver is 8.8.8.8 hosted by Google.

If a name is not available in either location, domain name resolution will fail.

For example there is a public website called web.mycompany.com. But if you work for this company and internally you wish to be able to refer to it as just web, you can do so by adding an entry into the `/etc/resolv.conf` with the `search` keyword. So for web.mycompany.com, add an entry `search mycompany.com`. Then if you ping web, it will automatically append mycompany.com to it and search for it, find it and resolve correctly. If you specify the domain in the query, meaning if you did a `ping web.mycompany.com` it would exclude including the search parameter mycompany.com to the DNS resolution request.

**DNS Management mapping***
A records - name to IPv4 mapping
AAAA records - name to IPv6 mapping
CNAME records - mapping a name to a name

`ping` may not always be the right tool to reach a website. `nslookup` does a lookup for a domain name in the DNS server, however it skips name resolutions from the /etc/hosts file. So any names in the hosts file will be ignored by nslookup. `dig` can also be used for DNS namespace lookup.

### Network Namespaces

Containers run on something called Process Namespaces which is a core Linux feature. What that basically means is that every docker container is a process (pid) running on a host machine. Inside that PID there are other processes running such that they are not aware of the parent PID or of the other PID's on the host. This in turn also ensures that PID numbers between the containers and the hosts never clash. It also allows for running multiple docker containers to run on the same host machine without PID clashes. The host machine has all the permissions to look into the PID's running under a particular PID.

Run `ps aux` on the host machine to get a list of all the running processes. Run `ps aux` in a container and you'll see a much smaller list of processes that do not coincide with the host. 

Similarly there are Network Namespaces for network level isolation. Containers run in their own network namespace, meaning that they have their own ARP and Route tables and do not see the network interfaces of the host machine. They may have their own virtual network interfaces. 

To create a new network namespace on a linux host, do `ip netns add <ns_name>` for example `ip netns add blue`. To get a list of namespaces, do `ip netns`.

Now in order to see the network interfaces attached with a host machine, we run the command `ip link`. In order to view the network interfaces for a network namespace such as the `blue` we created earlier, use the command `ip netns exec blue ip link` or the command `ip link -n blue`. This will only show the loopback interfaces and not the eth entries.

Similarly for the arp tables. `arp` shows entries on the host level;, while `ip netns exec blue arp` runs it in the container namespace called blue.

The same is applicable for routing tables as well. `route` for the host machine routes while `ip netns exec blue route` for the routes defined on a container level.

To connect 2 network namespaces:

```bash

#1. Create a connection between 2 virtal ethernet ports of type veth
$ ip link add veth-red type veth peer name veth-blue   # Create the connection pipe and include the interfaces that it will be connecting.

#2. Link the virtual ethernet ports to the correct network namespaces
$ ip link set veth-red netns red   # attach the virtual interface veth-red to the network interface red
$ ip link set veth-blue netns blue   # attach the virtual interface veth-blue to the network interface blue

#3. Assing IP addresses to the virtual ethernet ports created on both the namespaces
$ ip -n red addr add 192.168.12.1 dev veth-red   # attaching the IP address to the red ethernet interface
$ ip -n blue addr add 192.168.12.2 dev veth-blue   # attaching the IP address to the blue ethernet interface

#4. Activate the network interfaces
$ ip -n red link set veth-red up
$ ip -n blue link set veth-blue up

#5. Ping the blue namespace from the red namespace to test
$ ip netns exec red ping 192.168.12.2   # ip address of blue being pinged from red ns.
```

***What does the network isolation look like?***

If you check the arp tables of the 2 namespaces as well as the host, you'll notice that the host has no idea of the network IP's and connectivity b/w the red and blue namespaces, however the red and blue namespaces have each other's IP's and MAC addresses in their respective ARP tables.

```bash
$ ip netns exec red arp   # will have an entry for the blue namespace IP address and MAC
$ ip netns exec blue arp   # will have an entry for the red nemepace IP addr and MAC
$ arp   # will bring up the host ARP details and will not have either of the namespaces's IP's
```


> This looks easy when you have 2 namespaces. When there are multiple namespaces to be connected, you need a virtual switch (bridge) on the host to do this virtual networking between the namespaces. You use softwares like Linux Bridge and Open vSwitch for this. 

To use a bridge, the following needs to be done. 

```bash
#1. Create a bridge ethernet interface on the host
$ ip link add v-net-0 type bridge

#2. It should appear in the list of ip-links available
$ ip link

#3. To bring it up (by default it is down at creation)
$ ip link set dev v-net-0 up

#4. (optional) if a connection already exists between 2 network interfaces that you're trying to connect, delete those interfaces as they will not make sense anymore, the network interfaces will be connecting via the new bridge now
$ ip -n red link del veth-red   # delete the red interface
$ ip -n red link del veth-blue   # delete the blue interface (this one actually gets deleted automatically since they're a pair)

#5. Connect the red network to the bridge
$ ip link add veth-red type veth peer name veth-red-br   # veth-red-br is the interface name on the bridge end
$ ip link add veth-blue type veth peer name veth-blue-br   # veth-blue-br is the interface name on the bridge end

#6. Now attach the veth-red interface name to the red network and the veth-red-br interface to the bridge network
$ ip link set veth-red netns red
$ ip link set veth-red-br master v-net-0

# and the same thing for the blue network

$ ip link set veth-blue netns blue
$ lp link set veth-blue-br master v-net-0

#7. Now attach IP addresses to the 2 networks

$ ip -n red addr add 192.168.12.1 dev veth-red
$ ip -n blue addr add 193.168.12.2 dev veth-blue

#8. Start the interface so the 2 networks are connected to the bridge
$ ip -n red link set veth-red up
$ ip -n blue link set veth-blue up

#9. To start connectivity between the 2 networks, the bridge in between also needs an IP so the machine knows where to route through. 
$ ip addr add 192.168.12.5/24 dev v-net-0   # assigns an IP address to the bridge 
```

In order for the bridged network connected above to have outside (w/ wo internet) connectivity, the network's routing table must have an entry for the gateway that will provide this access.
The gateway is a host connected to this whole network we setup earlier and also connected to the other network (or internet). Hence this hsot has multiple IP addresses, one in the virtual network we just created and one ethernet connection to the outside network. We use the gateway's IP on the bridged network side in the routing table to route traffic outside of the private network. 

```bash
#10. The gateway's IP address on the private network is assumed 192.168.12.5
$ ip netns exec blue ip route add 192.168.12.0/24 via 192.168.12.5
```

> At this point, the outside network will be reachable but the responses from the external servers will not come back because there is no NAT at the gateway, hence the gateway doesn't know how to route messages back to the individual hosts making the call. We need to setup NAT such that the Gateway server replaces the FROM details from the outside response and puts its own name and address instead, and sends out the requests to external servers with it's own IP address so the internal network is not exposed to the outside world.

```bash
#11. Setting up NAT on the gatway node
$ iptables -t nat -A POSTROUTING -s 192.168.12.0/24 -j MASQUERADE

#12. Setup the gatway node as the Default route on the routing table so it can reach the internet from the gateway
$ ip netns exec blue ip route add default via 192.168.12.5
```

To facilitate traffic moving from the public internet into the private network, for example, to access a website hosted on a machine on the private network, we can either expose the IP address of the private machine or setup a port forwarding rule on the gateway host. 

```bash
#13. Forward all requests on the port 80 of the gateway node to port 80 of the machine 192.168.12.2 in the private network
$ iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.12.2:80 -j NAT    
```

## Docker Networking

Docker containers can have the following network options:

1. None: Connect to no network, no connectivity to the outside world at all.
2. Host: Use the hosts ethernet network, meaning that if the container exposes an app on port 80 and the networking is set to host, the app will be exposed on port 80 of the host. 
3. Bridge: By default docker creates a network on a host called a bridge network, and all the containers on that host are connected to that bridge network. This is the default networking mode of containers if nothing is specified. This interface for this network is called `docker0` on the host machine. 

Internally, docker creates a new network namespace for each container and connects them using the bridge network.

```bash
$ docker run --network=host --name my-contianer <image-name>
```

## CNI - Container Network Interface

All implementations, wether it be docker, mesos, k8s, rkt or network namespaces, work on the same fundamental steps:

1. Create a network namespace
2. Create a bridge network/ interface
3. Connect veth pairs (pipes)
4. Attach veth to namespace
5. Attach the other veth to the bridge
6. Assign IP addresses
7. Bring the interfaces up
8. Enable NAT - IP masquerade

Steps #2 to #8 can be abstracted out into the single approach instead of coding it all over again for all platforms (k8s, mesos, rkt etc.).

CNI comes into to setup a standard interface (or mechanism) by which programs that abstract out #2 to #8 should be developed to solve these network challenges in a container runtime environment. The programs are called plugins. CNI defines how these plugins should be developed and how container runtimes should invoke them.

Multiple network solutions that are CLI compliant are available open:
- bridge
- vlan
- ipvlan
- macvlan
- windows
- DHCP
- host-local

Other 3rd party solutions are:
- VMware NSX-T
- Calico
- Weaveworks
- flannel
- cilium

> docker however does not implement CNI, but implements its own set of standards called CNM (Container Network Model). That means that docker doesn't natively work with these CNI plugins.

```bash
$ docker run --network=cni-bridge nginx   # XXX this will not work because docker doesn't implement CNI

# Instead you'll have to create a docker container without any network interface and then use the CNI tool (using bridge in the example below) to connect it to the network
$ docker run --network=none nginx
$ bridge add <container-id> /var/run/netns/<network-namespace-id>
```

This is how kubernetes runs docker containers and supports the use of any CNI conformant plugins as well.

## Network Setup for k8s nodes

- The nodes must be connected to one common network bridge so that they can communicate with each other
- Every node must have a distinct IP address, machine name and a MAC address.
- Additionally certain ports also need to be open between nodes in the cluster for the control plane to communicate. 

**Master**
- kube-apiserver: 6443
- kubelet (optional/ if you're running app containers on the master): 10250
- kube-scheduler: 10251
- controller-manager: 10252
- etcd: 2379
- etcd clients (optional/ required only in multi-master clusters so the etcd clients between 2 masters can communicate): 2380 

**Worker**
- kubelet: 10250
- services: 30000-32767 (non elevated ports available for apps to run on)


## Container Native Interface

CNI is the specification that defines the structure for defining networking in k8s. This spec is used by all K8s netowrking solutions including flannel, calico, NSX-T etc. The 3 requirements for networking in k8s are:

1. Every pod should have its own unique IP address
2. All pods on a single node should be able to communicate with every other pod on the same node
3. All pods should be able to communicate with every pod on other nodes without using NAT

The kubelet on every node is responsible for running the pods. It uses the `--cni-conf-dir` to look for the name of the script that is to be run. This script is responsible for giving an IP address to every pod, attaching it to the bridge network, creating the veth pair and bring up the interface. It is also responsible for deleting the veth pair as well as freeing up IP addresses. 
It then looks into the `--cni-bin-dir` to find the script to run and then executes it with the `add` command specifying the container and the name and namespace of the container.

Inside a cluster, the `/opt/cni/bin` has all the supported CNI plugins as executables. 

If there are multiple configuration files in the `/etc/cni/net.d` directory, it will select the one that comes first in alphabetical order. 

The defined CNI plugin responsibilities are:
1. Must support arguments ADD/ DEL/ CHECK
2. Must support parameters container id, network ns etc.
3. Must manage Ip address assignment to pods
4. Must return results in a specific format

### Useful commands when working with Networks

- List Network interfaces - `ip link`
- Get status of a particular network interface - `ip link show <interface-name>`
- Get list of services and the ports they are listening on - `netstat -tnlp`
- Check amount of traffic on a particular network port for a particular service - `netstat -anp | grep etcd`
- Get details for a running service such as kubelet - `ps -aux| grep kubelet`
- Show the IP address range for a particular network interface - `ip addr show <network-name>` - Example: `ip addr show weave`
- Show the route out of a node, including the gateway etc - `ip route` or `route -n`


### Weaveworks

Weave CNI deploys an agent/ service on every node that communicate with each other an share information around nodes/ pods etc. Weave also sets up the bridge network on every node called 'Weave', as well as the IP addresses. 

> Containers may be a part of multiple networks. Weave ensures that the correct network is selected to route the data out of the pod. 

When trying to transmit data from one pod to another, sitting on different nodes, the source pod sends the data to the destination pod. The weave agent on the source node intercepts the outbound request and encapsulates the data, replacing the source and destination addresses, and then transmitting it to the destination node. The agent on the destination node will then decapsulate the message and then push it to the pod it is meant for.

Weave can be installed directly on the nodes, or as pods on the cluster. When installed as pods, it is setup as Deamon Sets. 

IPAM: IP Address Management - How are the IP's allocated to the bridge networks and the Pods etc in the cluster, where the information is stored and who registers IP to resource mapping. 

CNI providers come with two components (DHCP and host-local) that help maintain pod IP addresses in a file. This is done by the `host-local` component. Host-local sets up a file on each host with manages the IP address of every pod. The ipam configuration that manages the subnet in which the pod IP addresses can be found under `/etc/cni/net.d/<conf-file>`, under the `ipam` -> `host-local` section. 

> Different CNI solutions handle this differently.

Weaveworks, by default assigns pod IP's in the 10.32.0.0/12 range, meaning a total over a million IP's for Pods. This is devided among the nodes. This can be configured using params passed during plugin setup.

## Service Networking

**Recap**: Services are exposed basically either internally or externally.

- For internally exposing services only to other pods within the cluster, use the service type clusterIP. This is not available outside the cluster, and other pods can reach the service using the clusterIp which is available across the cluster on all nodes.
- For exposing a service externally, the basic type is a NodePort. In this, first a clusterIp is created for the service, and then a random port on the host is forwarded to the service port. The same externally facing port is used on every node. 

Note: Load balancing sits on top of the NodePort by balancing the load arcoss the exposed port on every node for a service.

**Basics**:
- kubelet runs on every node and speaks to the API server and creates pods on every node
- kube-proxy runs on every node and checks with the API server to create services. Services are created across the cluster and not on a specific node.
- Services are a virtual object and do not exist as processes etc on a node or namespace.
- Services object is assigned a defined IP from an IP range. The kube-proxy component on each node gets the Ip address and creates the forwarding rules on each node to redict traffic on the node to the pod's IP address. Note that forewarding is on the basis of IP and port combination.
- kube-proxy may use userspace, ipvs or iptables for defining the forwarding rules. By default iptables are used.


**Assignment Steps**
- pod, on creation gets an IP address.
- when it is exposed using a service, the service gets an IP (cluster IP). This is from the IP range defined on the api-server with the field `--service-cluster-ip-range` which defaults to `10.0.0.0/24`. This can be checked using `ps aux| grep kube-api-server` as the startup parameter `service-cluster-ip-range`.

Check the rules setup for a service in the iptables using the command - `iptables -L -t net | grep <service-name>` . The service name is defined as a comment on every service. It will also show tcp dpt (destination port) which is the port exposed by the service, and a DNAT rule to redirect the tcp traffic to, below the cluster IP entry, depicting the pod cluster IP and port to which the service will target the service output to, on that machine.

kube-proxy also makes an entry for all this in the kube-proxy logs at `/var/log/kube-proxy.log`


## Kubernetes DNS

Whenever a service is created, kube-dns creates a record with the service name and maps it to the service's ClusterIP address. For example, if you're trying to access a pod called web (exposed using a clusterip service called web-service) from a pod called test, 

- If test is in the same namespace as web, you can directly access http://web-service
- If test is a different namespace from web, say web is in app and test is in common, then test can call web as http://web-service.app

Beyond this, for the FQDN of a service the mapping is as 

hostname (web-service) --> namespace (app) --> Type (svc) --> Root (cluster.local)

Hence, the FQDN for the web-service service would be http://web-service.app.svc.cluster.local using which it can be accessed throughout the cluster.

Note: FQDN's are not generated for pods by default, it needs to be enabled. Once enabled, the pod name is not used as the hostname, but instead the pods IP is used (dots replaced by dashes), to create the FQDN. Hence for a pod with the IP address 10.244.10.5, deployed in the apps namespace, the fqdn would be http://10-244-10-5.app.pod.cluster.local

Kubernetes deploys kube-dns as a pod, as the dns solution prior to 1.12. 1.12 and beyond, it implements coreDNS. The configuration file for coreDNS is called a corefile at `/etc/coredns/Corefile`. This uses plugins, including one for kubernetes. The root location for all dns entries (cluster.local) is defined here. The nameserver used by the dns is in the `/etc/resolv.conf`. Inside a cluster, describe the configmap to view its properties. 

By default, DNS entries are not created for PODs (by default it is set to insecure) , but it can be enabled using the `pod` property in the codedns corefile. The config file is injected into the pod as a configMap.

When we deploy the coreDNS in the cluster, it is exposed as a service called kube-dns in the kube-systme namespace. This is what pods use to resolve DNS entries. The setup of the DNS entry for the DNS server is done by the kubelet in the `/var/lib/kubelet/config.yaml` file, under the ClusterDNS entry.

nslookup and the host commands also show the IP for the service as well. `host web-service` for example. The host command works because the reolv.conf also has a search entry for `default.svc.cluster.local`, `svc.cluster.local` and `cluster.local`. Note that the search entries are avilable only for services and not pods by default. To search for pods as well, the full FDQN of a pod needs to be provided such as `host 10-244-10-5.default.pod.cluster.local`



## Ingress

### Services vs. Ingress

Services like ClusterIp and nodeport allow you to expose your services internally or externally in the cluster. If using load balancers external to the cluster, using cloud provider LoadBalancers, you'd create a cloud provide loadbalancer everytime you created a service type LoadBalancer. These loadbalancers would then need another global load balancer on top of them for Layer 7 routing. 

In order to avoid this situation, you can use the ingress controller inside the cluster which acts like a layer 7 LB inside of your cluster. Note: This Ingress is internal to the cluster and this also needs to be exposed to the outside world as a nodeport or a cloud provider LoadBalancer. This however, would be a one time configuration. The tool deployed inside your cluster that does this ingress, like an HAproxy, Trafeik or nginx, are called the Ingress Controller, and the configuration for routes and applications is called Ingress resources. 

> BY DEFAULT, KUBERNETES CLUSTERS DON'T COME WITH AN INGRESS CONTROLLER.

Ingress solutions are :
- GCE (Google's solution)
- Nginx
- Contour
- HAproxy
- Istio

> GCE and Nginx are currently being supported and maintained by Linux Foundation.

These are not just LB's but also have intelligence to automatically add resources into them as well. Nginx is also deployed as a deployment on your cluster, fronted by a NodePort service.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  labels:
    tier: web
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image:  quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
          - /nginx-ingress-controller
          - --configmap=$(POD_NAMESPACE)/nginx-configuration
          env:
          - name: POD_NAME
            value: nginx-ingress-controller
          - name: POD_NAMESPACE
            value: default
          ports:
          - name: http
            containerPort: 80
          - name: https
            containerPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
  selector:
    name: nginx-ingress
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

But in order for the ingress controller to automatically enroll new pod instances for the load balancing, it needs a ServiceAccount which it uses to control k8s resources. The serviceAccount needs to be created with the correct Role and RoleBindings.

The part of the ingress controller that defines all the routing rules is called the ingress resource. 

In summary, the components used to run and configure an ingress controller to loadbalance basis defined rules, the components required are:

- nginx-configuration - the configuration file for the nginx-ingress-controller, deployed to a particular namespace.
- nginx-ingress-deployment - the actual nginx-ingres-controller deployment which deploys the nginx pods in the namespace
- nginx-service - NodePort service to expose the nginx deployment pods
- nginx-serviceaccounts - to provide access to the kubernetes resources to nginx pods
- nginx-resource - can do routing based on domain name or URL

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: <resource-name>-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:   # Not mandatory, required only if configuring tls
  - hosts:
    - <hostname>
    secretName: <secret-name-containing-certificate>
  backend: # for a basic routing to backend without any rules
    serviceName: <app-service-name>
    servicePort: <app-service-port>
```

Let's assume a scenario where we want to setup the following configs

Configuration for routes under the same domain:
- www.mywebsite.com
  - /wear
  - /store

Configure for subdomains
- www.wear.mywebsite.com
  - /categories
  - /reviews
- www.store.mywebsite.com

> So, an ingress controller can be used on routes on the same domain an subdomains, but can the same ingress resource be used to redirect to different domain names?

> Seems like the default backend service cannot be changed easily. You have to create a backend service in the name specified by the ingress description `kubectl describe ingress <ingress-name>`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-resource
spec:
  rules:
  - http:
    paths:
    - path: /wear
      backend:
        serviceName: wear-service
        servicePort: 80
    - path: /store
      backend:
        serviceName: store-service
        servicePort: 80
```
For configuring subdomains:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-resource
  annotations: nginx.ingress.kubernetes.io/rewrite-target: /   # Described below, nginx specific feature
spec:
  rules:
  - host: wear.mywebsite.com
    http:
      paths:
      - path: /categories   # Notice, the subdomain also has a path under it specified explicitly
        backend:
          serviceName: categories-service
          servicePort: 80
      - path: /reviews
        backend:
          serviceName: reviews-service
          servicePort: 80
  - host: store.mywebsite.com
    http:
      paths:   # Notice, the path property is missing because we want all subroutes under store to continue routing to this service only
      - backend:
        serviceName: store-service
        servicePort: 80
```

The nginx-rewrite property is what ensures that the url route is not pasted at the end of the destination route. For example:

Without nginx rewrite-target:

`https://<ingress-ip>.<port>/watch --> https://<watch-service>:<service-port>/watch`

But since the `watch` path is not present in the actual application, it shouldn't really be there at the end of the URL.

With nginx rewrite target:

`https://<ingress-ip>.<port>/watch --> https://<watch-service>:<service-port>/`