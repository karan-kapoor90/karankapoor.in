---
layout: post
categories: [Writeups, Kubernetes]
comments: true
excerpt: A primer guide for Secuirty in Kubernetes
title: Kubernetes Security
tags: [kubernetes, cloud native, security, RBAC, Authentication]
description: A primer guide for Secuirty in Kubernetes
---

## API Server Access

K8s doesn't have a native resource for users. They can instead be used from a csv file which may have a clear text password for users or a encoded password. The format for the file is as follows:

```bash
<password>,<username>,<random-user-id>
password,karan, u0001
password,,user1,u0002
```

Then edit the kube-apiserver yaml file and pass in the the csv file in the container spec. For a cleartext password file, add the command `- --basic-auth-file=/path/to/file/filename.csv`. If you want to create a csv file with a token inside it instead, the command to add would be `- --token-auth-file=/path/to/file/filename.csv`

Then you must create Role and a Role Binding for these users. 

```yaml
apiVersion: rbac.authorizarion.k8s.io/v1
kind: Role
metadata:
  namespace: my-namespace
  name: pod-reader
rules:
  - apiGroups: [""]  #  [""] indicates core API groups
    resources: ["pods"]
    verbs: ["get","list","watch"]
```

In order for the users to be binded to this role, the rolebinding would be as follows:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: my-namespace
  name: pod-reader-binding
subjects:
  - kind: User
    name: user1
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role  # this muse be Role or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

You can then make a REST API call to get the pods by:

```bash
$ curl -v -k https://<k8s-ip>:6443/api/v1/pods -u "user1:password"
```

## TLS within the cluster

Lets say company A has a key pair and needs to publish his public key for public usage (aka ssl on his web site). 

- Company A must make a certificate request (CR) to a certification authority (CA) to get a certificate for his key pair.
- The public key, but not the private key, of company A's key pair is included as part of the certificate request.
- The CA then uses company A's identity information to determine whether the request meets the CA's criteria for issuing a certificate.
- If the CA approves the request, it issues a certificate to company A. In brief CA signs company A's public key with his(CA's) private key, which verifies its authenticity.
- So company A's public key signed with a valid CA's private key is called company A's certificate.

Now, here's how a 2 way SSL happens. Here, the client is say your web browser and the server is say google.com.

1. You open google.com and request for something.
2. Google has a public and private key pair. It must now send you the public key so you can use that public key to encrypt your login details. Essentially, your login details are first encrypted by you using a session key which is a random key you decide on, but must share with the server so it can use that to decrypt the data you shared. This whole packet (login data + session key used to encrypt said data) is then to be encrypted using the public key sent to you by the server in the certificate. The certificate is a signed certificate (signed by a CA that your browser recognises as a valid certifying authority).
3. Your browser runs the certificate sent by google through the CA which validates the signature and approves if the certtificate presented to you by google is actually a google certificate. 
4. Once the CA says the certificate is authentic, you pull out the public key from the certificate. 
5. You encrpyt the login data using the session key and then encrypt the whole packet including the session key, using a the public key and send the whole thing to the server.
6. The server now checks the client certificate with the CA to see if that is valid. Note that the client certificate is only used for verifying the clients identity. It isn't actually used to encrypt anything. 
7. The server has now established the authenticity of the client as well. It encrpyts the response to the clients requested information using the session key sent by the client in #5. 
8. The session key is established as the encryption key for this communication session successfully.  

### Server certs

1. kube-apiserver: This exposes any/ all of the k8s services / REST API's outside over https. It does this using the public key - apiserver.crt and the private key apiserver.key.

2. etcd server: stores all the information about the cluster

3. Kubelet: interacts with the API server for worker node management


### Client certs

The difference between the username/ password vs. using client certificates is that when you use a username and password, anybody who manages to get their hands on your passowrd can authenticate themselves as you and the server will trust them, because it is authenticating you by using the username and password combination to do that. Client certificates identify your machine to the server instead of you. You could hence put a certificate on your machine that is sent to the server with which the server identifies you. You could specify the key to use to authenticate you with the server in the case where for example you're using the curl command. The ssh command or connecting to a git repo using the ssh key instead of https is an example of using client certificates for authenticating the client. Note that you could use a combination of both ssl keys + username/ password and this is called Two Factor Authentication. 

Username/ Password based auth is called basic auth, certificate based auth is called Certificate Based authnetication. 

Client certs are required by agents/ resources that need to talk to a server component, to verify their own authenticity. The client certificate helps the server validate that the user is who he claims to be.
Since the kube-apiserver is a central piece for external and internal actors to talk to other components in the cluster, the following clients need client certs to talk to the apiserver.

1. Admins
2. kube-scheduler
3. Kube-proxy
4. kube-controllermanager

Hence it is required for the cluster to have a signing authority in the cluster also. You could even have 2 CA's. One for the etcd server and it's certificates and another for all the other cluster components. 

### The process of securing

On a broad level, there are 3 parts to this

1. Generating the CA certificate which will be used to sign client and server certificates. The CA needs to have its own private key and certificate. This is what defines that the CA is legitimate. There are multiple utilities like openssl, easyrsa, cfssl etc. Here's what you do:

- Generate private key

```bash
$ openssl genrsa -out ca.key 2048
```

- Generate a certificate signing request. This has all the details but just not the sign. 
```bash
$ openssl req -new -key ca.key -subj "/CN=kubernetes-ca" -out ca.csr
```

- Sign the certificate. Since this is for the CA itself, it is self signed. 

```bash
$ openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

> The cert names may be different for different clusters.

For all other certificates, we will use this key to sign them.

2. Client certificate. If trying to generate the certificate for a user, do the following:

- Generate the private key

```bash
$ openssl genrsa -out admin.key 2048
```

- Create the certificate signing request

```bash
$ openssl req -new -key admin.key -subj "/CN=<username>" -out admin.csr

```

> This can be for the admin-user or any other user you like. There is no such limitation.

- Generate a CA signed certificate

```bash
$ openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

> Note that the groups to which the user would belong would also be based on the certificates. A user's group can be specified while creating the Certificate Signing Request in the subject using the /O parameter as `opessl -req -new -key karan.key -subj "/CN=karan.kapoor/O=system:masters" -out karan.csr` . system:masters is a k8s group.

3. Server certificates. In order for the server to respect and identify the CA as an authorized one, the CA's root certificate needs to be available to the server components.  

Taking an example of the kube-api server, everyone talks to the kube-apiserver and for multiple reasons it may be references using various different names such as simply kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster.local, master node IP or the docker container's IP where it is running. The point is, all of the names that may be used by applications or admins to interact with the API server needs to be avialble in the certificate otherwise the requets will not be accepted. This needs alot of names to be configured for the server and can be done with the help of a cnf configuration file where we'll give all the alternate names for the service.

- Generate the private key

```bash
$ openssl genrsa -out apiserver.key 2048
```

- Create the certificate signing request, but pass the configuration for the various names for the server

```bash
$ openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf

# the openssl.cnf file looks like this

[req]
req_extensions = v3_req
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.0.1.10
IP.2 = 156.24.23.10
```

- Sign the certificate

```bash
$ openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt
```

**In case of the API server**

The API server acts as both, a client when talking to the etcd cluster as well as a server because the other components talk to it directly. Hence it must have all the client configuration as well as the server configuration. All of this configuration is passes in as parameters to the kube-apiserver process and can be found in the kube-apiserver.yaml file definition. This is what needs to be updated when the certs are changed. 
For example, in the kube-apiserver service definition:

```bash

# if you looked for the parametrers passed to the API server process, you would find these amongst other parameters there.

# Keys for when the API server is the server - or accepting requests from the outisde world
--client-ca-file=   # path to the CA server cetificate file for identifying the CA
--tls-cert-file=   # path to the API sever certificate (public key)
--tls-private-key-file=   # path to the private key file for the API server.

# Keys for when the API server is acting like a client - or communnicating with the etcd cluster
--etcd-ca-file=   # path to the CA server certificate that has issued the CA cert for the etcd cluster. Note that this may be a CA different from the CA that signs the keys for all the other components in the cluster
--etcd-certfile=   # public key of the client certificate pair for the apiserver
--etcd-keyfile=   # private key of the client certificate pair for the apiserver 

# There are similar fields to define connectivity to the kubelet server as well.
```

**In case of the kubelets**

Kubelets are https servers that run on every worker node. These talk to the API server and the vice versa as well. This means that both client as well as server certificates need to be created for these.

***Server Certificates***
The certificates are issued to the name of the node and not a common generic name etc. for the CN in the certificate definition. They are named after the node.
Every node will have its own kubelet config as follows: 

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  x509:
    clientCAFile:   # path to CA certificate file
tlsCertFile:   # path to TLS server certificate
tlsPrivateKeyFile: # path to TLS private key
```
***Client Certificates***
The kubelets must also have a client certificate that needs to be generated for the server to identify the kubelet and give it the right permissions. These client certificates are obviously on a per node basis and use the name and address of the node as well for node identification.

Since the nodes are system components like the controllerManager etc, the name on the cert must be int he right formart. The right format is `system:node:nodename`. The group name should be `SYSTEM:NODES`.


### Using the client certs

You could use the certs in a REST API call to the server as follows:

```bash
$ curl https://kube-apiserver:6443/api/v1/pods --key admin.key --cert admin.crt --cacert ca.crt
```

This is usually done by creating a config file though 

```yaml
apiVerison: v1
kind: Config
clusters:
- cluster:
    certificate-authority: ca.crt
    server: https//:kube-apiserver:6443
    name: my-cluster
users:
- name: kubernetes-admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

### Debugging Certificate issues

Where the certs for a cluster are located depends on how the cluster was setup.

1. Using kubeadm - `cat /etc/kubernetes/manifests/kube-apiserver.yaml`
2. The hard way, as a system service on the master node - `cat /etc/systemd/system/kube-apiserver.service`

It makes sense to go to these locations and pick up the certificate locations and keep them iin an excel file so you're always on top of them. You'll need to decrypt the certificate using `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout` . This command will output the response to sysout for you to look through. 

If you've setup your ***cluster components as a service***, look through the logs as 

```bash
$ journalctl -u etcd.service -l
```

***using kubeadm***

```bash
$ kubectl logs etcd-master
```

> Sometimes because of the security certificates, if the kubectl tool fails, the debugging can be done at the docker level.

```bash
$ docker ps -a   # get the container ID from this
$ docker logs < container-id>   # get the logs for that particular container
```

> Using the aforementioned mechanisms, one can create keys and certificates for users as well as applications to connect to k8s with. Imagine doing this for every new admin/ user or for rotating cluster certificates every now and then! A more simpler and graceful solution is to use the CertificateAPI in kubernetes.


## Certificate API

The certificate API is a resource in kubernetes when your cluster is created using kubeadm. This allows users to send a certificate signing request to the cluster and allows the admins to gracefully manage, approve and reject cert signing requests using the kubernetes api directly.

How this works is as follows:

1. The user creates a CertificateSigningRequest Object

- user generates the key
```bash
$ openssl genrsa -out jane.key 2048
```

- User creats a certificate signing request and shares it with the Admin
```bash
$ openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
```

2. The Admin reviews the request

- The admin creates a CertificateSigningRequest kubernetes object from the key (CSR) shared by the user.

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
      <convert the file received from jane, jane.csr to base64 and put the contents here without any quotes etc. Convert using cat jane.csr|base64>
```

This can then applied into the environment. Admins can view these requests using the following command:

```bash
$ kubectl get csr
```

3. Admin approves the request

Approve using the following command

```bash
$ kubectl certificate approve jane   # conversely, the command to reject is the verb deny instead of approve
```

4. Shares the certs created, with the user

```bash
$ kubectl get csr jane -o yaml
```

In the yaml body, take out the base64 encoded certificate, decode it and share it with the user.

```bash
$ echo "<encoded certificate text from the yaml>" | base64 --decode
# or just export it to a file so you can share it with the user directly
$ echo "<encoded certificate text from the yaml>" | base64 --decode > jane-signed.crt
```

> This process of signing and approving the certificate requests is done using a component in the Controller Manager in the control plane. The controller manager has 2 components inside it called the "CSR-APPROVING" and "CSR-SIGNING" that make this functionality possible. The CA certs required to sign the key are a part of the Controller Manager's yaml configuration in the cluster-signing-cert-file and cluster-signing-key-file attributes.


```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: akshay
spec:
  groups:
  - sytem:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
      LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZV3R6YUdGNU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQXNDL25iaFFaYUFTOWNEUGs4WFpuV0g1V1h3Snp1T0ErckwyYXI0MnRIWGI2CnpnbWdSL1ZQT3ZjaVJ1K1V0dEdRMHVLYVNNT0YwL2lpQ3h1MGowMW9mNVFRVXRrMUx4MTYwZXJIYm1JazFacm4KRnVkakVKTElSV1YzOEhZS3Q1MTR2blZ1ek52ZWxUSGE5RHdzSzQ1NEk4T1lnUkhJZWZMMW9jU3RIYTdVaGRHRwpjQ1dlc2ZGdlg5QXl6ZVg3bEJvR0hLaktvLzlaY3JGeWpsNGRKdzgzV2REUnZQcEI2Mkt2ekxSaE9Ha091VEVlCldyNHlqMjNRLzJYb0REQzA4NlYrNFRvVU1hWWd5WXpHK2Z3NVlBVGxpdUFBT3FHaGJUYmJYU1ZsenFVSmlXbisKTlFiZHNUMFVJekVHZ0xBc3YxbDNCZ3ZiSWdCTFN4eSt6eW1PbnZuUVVRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBSW1nU2VRMUNOWUkyU1MwT1dFUnBJSkdZQjRvb2FJQ3JFRXBDcUJFWnh5YmVNeVFuQUJDCjQ3eGlUVzlyaFE1czZSOUE2aDhweTkxK1NiZkhPY0k1WTVhK0E5NmdyNXI2b0Y0Y1dMb2d0amFvcHViQkt6QlQKYkw0b2dQYitsQUgzbnBhcnhDcng4ZW14cGFlbzE5Sk8zVGpWTHZIY0lLSzFuOFhCdHNUSStIYUdUMDhUbGVCKwp0SVNlQ29LaGNWUzNIMkUza2l3VVRBVjF6YlpVakdsQndPcitnZFZrSnM3VEUrYWN2SjZ0aGxGU0dkZTJFYnRFCkRaYVBqYWQ3NVBKNHNuRGlUVEhmRWJhTG1VYVo4TmcvdlVYYy9McHdnZk05c3NmRXZZNzhhd2ZnRU0xdWthdnYKZzN0VFFDTVIvWE9WM2pRL2M5VzlBOFdQUzFEclNrSDBMVzg9Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
```


## Kube config

In order to access a cluster, if using curl, you'd need the apiserver URL and port, the user's key, certificate and the CA's certificate like so

```bash
$ curl https://<api-server-ip>:6443/api/v1/pods --key <user-key-file> --cert <user-certificate-file> --cacert <ca-crt-file>
```

. Internally kubectl also needs those things but abstracts that into the kube.config file in the user's home directory - `$HOME/.kube/config`

- Clusters

This is basically all the clusters you have access to. This could include some local clusters, some cloud managed clusters etc. 

- Users

These are the various users that may have varied levels of privileges. For example you may be an admin (user called admin) on your own local cluster but may have only developer (user called developer) permissions on an organizational cluster. The user in itself is not tied to a particular cluster mentioned earlier. 

- Contexts

A context is what brings the users and the clusters together. For example you could have the same user Admin who can administer both a local as well as a GKE cluster. Hence Admin@local and Admin@gke-prod. In this case you're not defining new users on the cluster, but instead using users that are pre-defined on the cluster. These users on the cluster have their own set of rights and privileges which is defined by the certificates etc we created earlier. 

Hence, the APIserver URL, port and CA certificate go into the clusters section, the user key and cert go into the Users section and the binding glue is the context section.

> current-context is the context you're currently running all queries/ kubectl commands against.

```yaml
apiVersion: v1
kind: Config
current-context: admin@myLocalCluster
clusters:
  - name: myLocalCluster
    cluster:
      certificate-authority: ca.crt
      server: https://localhost:6443
  - name: gcpCluster
    cluster:
      certificate-authority: gcp-ca.crt
      server: https://gcp-url:6443
users:
  - name: admin
    user:
      client-certificate: admin.crt
      client-key: admin.key
  - name: developer
    user:
      client-certificate: dev.crt
      client-key: dev.key
contexts:
  - name: admin@myLocalCluster
    context: 
      cluster: myLocalCluster
      user: admin
      namespace: default   # optional
  - name: developer@gcpCluster
    context:
      cluster: gcpCluster
      user: developer
      namespace: finance   # optional
```

> This file doesn't need to be created/ applied on a cluster like deployment etc. other resources. This is used by the local kubectl on your machine (or the machine you're using to interact with the cluster) to decide which cluster to talk to.

```bash
$ kubectl config view   # to view the config file in your environment
$ kubectl config view --kubeconfig=my-custom-conf   # to point to a custom configuration
$ kubectl config use-context developer@gcpCluster   # to change context to the developer user on the gcp cluster. This will be updated to the current-context as well.
```

> Instead of giving the path to the certificates and keys, you can also insert them into the file itself after base64 encoding. Replace the `certificate-authority` field with `certificate-authority-data` and similarly, the `client-certificate` with `client-certificate-data` and `client-key` with `client-key-data` respectively. Simply convert the crt and key file data into base64 using `cat <filename>|base64` and paste the contents into the fields suffixed with data.

## API Groups

`/healthz`, `/api`, `/apis`, `/metrics`, `/version` etc are all API endpoints exposed by Kubernetes. From a resorces POV, as in the deployments, services, pods etc., the interaction primarily happens with the `/api` and `/apis` endpoints only. 

Endpoints under `/api` are called core API's and the endpoints under `/apis` are called Named API's. The newer resources being added to k8s are all named api's. Simple things like configmaps, secrets, pods, replication controllers, bindings, events, nodes, PV, PVC, services etc. are all core API's and come under that head. Resources such as Replication Sets, Deployments etc. 

Resources under `/apis` form groups such as:

- /apps
  - /v1
    - deployment
    - statefulsets
    - replicasets
- /extensions
- /networking.k8s.io
  - /v1
    - /networkpolicies
- /authentication.k8s.io
- /storage.k8s.io
- /certificates.k8s.io
  - /v1
    - /certificateSiginingRequests

The ones at the top such as /apps, /extensions, /certificate.k8s.io are called API groups, and the deployment, networkpolicies, certificatesigningrequests are called resources.
Each of these resources have their own set of verbs that can be used to action on them, such as:

- list
- get
- create
- delete
- update
- watch

## RBAC

Role Based Access Control allows you to create Roles that can then be used to restrict or grant access to users. 

### Roles and Role Bindings - Namespace Scope

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: my-ns   # optional. Not specifying will apply this object to the default namespaces
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list","get","create","update","delete"]
  resourceNames: ["blue", "green"]   # optional. For example this will allow the user to have access to only pods green and blue.
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

For core groups, apiGroups can be left empty. 

To link a user to a role, you need to create a RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: default   # optinal
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Apply using the usual `kubectl create` command.

### Impersonating a cluster user / checking your permissions

You can check what your role is allowed to do or impersonate another user to check their level of permissions (for example if you're an admin checking the privileges of a user). 

```bash
$ kubectl auth can-i create deployments   # responds with a yes or no
$ kubectl auth can-i delete nodes   # responds with a yes or no
$ kubectl auth can-i get create certificatesigningrequests   --as dev-user   # impersonates dev-user
$ kubectl auth can-i create pods --namespace production --as dev-user   # impersonate dev-user and check if he has the permissions to create pods in the production namespace.
```

> For a cluster wide RBAC policy, we use something called ClusterRole and ClusterRoleBinding. 


> Roles and RoleBindings apply to namespaced resources. For non-namespaced (cluster scoped) resources like nodes, namespaces, certificateSigningRequests etc., one has to use clusterRoles and clusterRolebindings. To view a comprehensive list of resources that are namespaced and cluster scoped resources, use the commands `kubectl api-resources --namespaced=true` . True is for namespaced resources while false is for cluster scoped resources.

### Roles and Rolebindings - Cluster Scope

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ClusterAdmin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verb: ["get", "list", "create", "delete"]
```

```yaml
apiversion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ClusterAdminBinding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: ClusterAdmin
  apiGroup: rbac.authorization.k8s.io
```

> You could use clusterRoles and their bindings for giving users cluster wide access to namespaced resources such as pods as well. In doing this, you grant access to a user to all resources of that kind across the whole cluster.


## Image Security

When you give the image name as just `nginx`, it assumes it to be `nginx/nginx`. If no registry is specified, it automatically assumes it to be `docker.io`. Hence, the image address is assumed to be `docker.io/nginx/nginx`.

To use another registry such as a private registry or a registry other than dockerhub, login to that registry using:

```bash
$ docker login <registry address>   # for example for google's registry it is gcr.io
```

The images are pulled on the nodes by a docker daemon on the node, hence the login credentials are required by the docker daemon. The credentials can be stored in a kubernetes secret as follows:

```bash
$ kubectl create secret docker-registry dockerregsecret --docker-server=<registry-address> --docker-username=<docker-username> --docker-password=<password> --docker-email=<email-ID>
```

> docker-registry is a built in custom secret type designed to store docker registry credentials.

In order to use these, in the pod definition file, the imagePullSecrets need to be specified. 

```yaml
apiVerison: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: mynginx
    image: privateregistry.io/my_reponame/nginx
  imagePullSecrets:
  - name: dockerregsecret   # this is the name of the secret we created earlier
```


## Security Context

Docker allows for you to add additional linux capabilities as well as to specify which user or group to run a container with. This capability is also available in kubernetes. You may choose to do this configuration on either the pod level or the container level on kubernetes. This can be done as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  securityContext:   # applied to all containers inside the pod; addidng capabilities not available on a pod level
    runAsUser: 1000
  containers:
  - name: myContainer
    image: nginx
```
or doing this on container level:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    securityContext:
      runAsUser: 1000
      capabilities: 
        add: ["MAC_ADMIN"]   # capabilities can only be added on a container level
```

> Note that configuration done on a container level superseeds that done on a pod level. Also, adding linux capabilities can only be done on a container level. 


## Network Policies

By default k8s requires all pods across all nodes to be able to reach each other through the ClusterIp internal IP of each pod, or service. By default, the policy is set to Allow all, meaning that even a web-server pod will be able to communicate with a DB server pod bypassing the App server pod. In order to ensure this doesn't happen, we have to create Network Policies which are Namespaced resources.

Essentially you set labels on the pod from which you wanna allow access and then create a network policy which allows access to a particular pod from a pod which has the selector lables.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dbpolicy
spec:
  podSelector:
    matchLabel:
      role: db   # which pods should the network polcy apply to. for example, this network policy will apply to any pod which is a DB pod
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
      matchLabels:
        name: api-pod   # which pods should the network policy be applied from. For example, this network policy will allow ingress to the DB pod from any pod that has the api-pod name label.
    ports:
    - protocol: TCP
      port: 3306   # port number which will be used for ingress into the DB pod from the API pod.
```

These network policies are enforced by the network solution deployed on the cluster hence not all network solution providers support policies. Some of those are:
- Kube-router
- Romana
- Weave-net
- Calico
- Canal

Network providers that do not support policies also exist, such as:
- Flannel

> Note, even if the network solution used does not support policies, you will not get an error message. The policies will simply not be enforced. 

## Service Accounts

Service accounts are used by applications deployed on top of the cluster to interact with the kubernetes API. For example - a jenkins server deployed as a pod on your kubernetes cluster will deploy resources to the cluster using a serviceaccount and not a User account (such as the one you use to interact with the cluster). Points to remember:

- Even when you don't mention a serviceAccount in a pod definition, a new service account is created in every namespace called default, which is injected as a volume in every pod deployed in the namespace. A describe of the pod or its yaml will show the same.

- In the case you want to inject a specific service account, in the Pod spec, it can be injected using the label serviceAccount with the service account name.

- In case you don't want to use a service account on the pod definition, you can specify `automountServiceAccountToken` to false in the pod spec.