## Various Roles

- Admins - Users
- Developers - Users
- Application End Users (Managed by App - Out of Scope)
- Bots - Service Accounts

k8s doesn't manage users directly. Have to use external systems.


## User Access

kube-api authenticates as REST and kubectl access

- Static Password File
- Static Token File
- Certificate
- Identity Provider

### Static Password File

Modify the kube-api options and add the option `basic-auth-file=user-details.csv`

```csv
password,username,userId01,groupOD01

## Example 

password01,user01,user01,developers
password02,user02,user02,developers
password01,admin01,admin01,admin
password01,operator01,operator01,operator
```

The kube-api server must then be restarted. If the installation is done using kubeadm, then the api-server file can be modified at `/etc/kubernetes/manifests/kube-apiserver.yaml`

For example, to authenticate to the REST API,

```bash
curl -v -k https://masternodeIP:6443/api/v1/pods -u "username:password"
```

### Static token file

Modify the kube-api options and add the option `token-auth-file=user-token-details.csv`

```csv
token,username,userId01,groupOD01

## Example 

kjbads6867927nbkdhasdh899ahd,user01,user01,developers
9878huh87h87h7hh98,user02,user02,developers
98j9j9879h89j,admin01,admin01,admin
98n98u96g76g6g,operator01,operator01,operator
```

The kube-api server must then be restarted. If the installation is done using kubeadm, then the api-server file can be modified at `/etc/kubernetes/manifests/kube-apiserver.yaml`

For example, to authenticate to the REST API,

```bash
curl -v -k https://masternodeIP:6443/api/v1/pods --header "Authorization: Bearer <token-value>"
```

> Static file based or static token based auth is not the prefered way of authentication.


## Service Accounts

`kubectl create serviceaccount <sa-name>`

 Creating a service account automatically creates a token which it then stores in a secret. The name of the secret is available when you run a describe on the service account. The secret account is linked is linked to the SA. This token can be seen using a describe on the secret and then be used as a bearer token for an application to used. 

The service account can be used in a pod/ deployment using on the same cluster by mounting the secret into the deployment. 

For every ns in k8s, there is a default SA in that ns. 

> Whenever a pod is created in the ns, the default secret token from that ns is automatically mounted into the deployment/ pod.

> The default secret is mounted to the pod at /var/run/secrets/kubernetes.io/serviceaccount - files inserted are ca.crt, namespace and token

> The default token in every namespace is fairly limited in terms of what it can do.

In order to make a pod use a service account different from the default token from the namespace, the service account can be mentioned in the pod definition yaml.

> A serviceAccount added to a pod, once added cannot be modified at runtime. The pod needs to be removed and recreated in order to change the serviceAccount. Instead, if mentioned in a deployment, the pod doesn't need to be deleted.

```yaml
apiVersion: apps/v1
metadata:
  name: my-pod
specs:
  containers:
  - name: my-container
    image: nginx
  serviceAccountName: non-default-sa
---

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
      serviceAccountName: non-default-sa
```

> You can choose not to automount the default service token account by using the field automountServiceAccountToken: false in the pod/ deployment spec



## TLS Certificates Basics

Users and components that use these to talk to each other. For example:
- Users are a client to apiserver
- kube-controllers are a client to apiserver
- kube-proxy is a client to apiserver
- kube-scheduler is a client to apiserver
- kubelet is a client to apiserver
- apiserver is a client to etcd
- apiserver is a client to kubelet

- server certs required for apiserver
- server certs required for etcd
- server certs required for kubelet

## Creating Certs

### Creating certificates for CA

Setting up the CA

- Create a Private key for the CA

```bash
openssl genrsa -out ca.key 2048
```

- Create a certificate signing request which will be used to create a certificate for the CA that will be signed using the key created in the previous step. This creates a cert signing request that then needs to be signed to create a public crt for the CA.

```bash
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
```

- Sign the csr using the key created earlier. 

```bash
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

### Creating an admin user

```bash
openssl genrsa -out admin.key 2048
openssl req -new admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
openssl req -in ca.crt -text -noout   # decodes and shows the contents of the certificate
```

### kube-apiserver cert creation

the kube-apiserver goes by various names internally, such as 
- kubernetes
- kubernetes.default
- kubernetes.default.svc
- kubernetes.default.svc.cluster.local

So in order to create the certificate, 

```bash
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf

# since we want to pass a bunch of DNS names to the csr creation command as a config file - openssl.cnf
---
[req]
req_extension = v3_req
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation,
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 19.96.0.10
IP.2 = 172.17.0.90
---

openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt
```

### kubelet server certs

The name on the certs is the node name and not 'kubelet'. This information goes in the kubelet config file - which is a kubernetes yaml file for the kubelet on every worker node. 

 There are client crts for kubelet also but this time instead of the node name as the user name, the name is `system:node:<node-name>` like `system:node:node01`, and in order for them to be given the right permissions, they need to belong to the group `system:nodes`. 


## Inspecting certs

In an environment setup using the kubeadm tool, look at the kube-apiserver.yaml manifest to identify the location of th certs. kube-apiserver.yaml can be found at `/etc/kubernetes/manifests/kube-apiserver.yaml`

Typically the certs are in the `/etc/kubernetes/pki/` folder or a subdirectory therein. 

To decode and look into a certificate, use the command `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout` 

The name under the subject section gives the username/ common name for the cert. The Alternative names are below and the validity and the issuer are also in the certificate. 

> kubeadm names the kubernetes CA as kubernetes

To start debigging - one must look at the logs. If the cluster services are setup as system services (when installing without kubeadm), use the `journalctl -u etcd.service -l` command to see the logs for the etcd system service. 

If the kube-apiserver or etcd are not running, we have to go one level down and look at the docker container logs and then using the `docker logs <container-id>` command to view the logs of the container.


# Kubernetes Certificates API
Admin creates a k8s resource called cert signing request. 

User creates a key --> send to admin --> admin creates csr (not k8s csr) --> Admin base64 encodes the csr --> Admin uses the encoded csr and creates a k8s csr resource --> admin gets csr in the cluster --> admin approves the request --> admin extracts the crt and gives it back to the user.

Jane:

```bash
openssl genrsa -out jane.key 2048
```

Admin

```bash
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
echo jane.csr | base64
```

Admin Creates a k8s csr resource 

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  groups:
  - system:authenticated
  signerName: kubernetes.io/kube-apiserver-client   
  usages:
  - key encipherment
  - digital signature
  - server auth
  requests: <base 64 encoded csr>
```

Admin must then approve the certificate after applying the csr.

```bash
kubectl get csr

kubectl certificate approve <csr-name>
```

Then the admin can extract the cert from the approved certificate by running a get on the csr in yaml

```bash
kubectl get csr <csr-name> -o=jsonpath={.status.certificate} | base64 -d > jane.crt
```


## KubeConfig

Has 3 sections:
- Clusters
- Contexts
- Users

Every request sent to the API Server needs the following information:
- Server IP/ Address
- client-key
- client crt
- ca crt


```yaml
apiVersion: v1
kind: Config
current-context: admin@my-cluster
clusters:
- name: my-clutser
  cluster:
    certificate-authority: ca.crt   # We can use certificate-authority-data field which has the ca.cert in base64 encoding
    server: https://<api-server-ip>:6443
contexts:
- name: admin@my-cluster
  context:
    cluster: my-cluster
    user: admin
    namespace: development
users:
- name: admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```