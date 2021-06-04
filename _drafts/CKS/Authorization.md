## API Groups

k8s has a bunch of API endpoints:

- /metrics
- /version
- /api - core group - core functionality --> /v1
- /apis - named groups
- /logs

Going forwards, the newer capabilities come under the `/apis` named group, such as 
- /apps
- /extensions
- /networking.k8s.io
- /storage.k8s.io
- /authentication.k8s.io
- /certificates.k8s.io

### Proxying the API Server

> if you tried to curl the API server for resources such as `curl http://IP:6443 -k` it would give you a 403 forbidden error

In order to talk to the kubectl REST API, you need to pass it the following params

```bash
curl http://IP:6443 -k --key admin.key --cert admin.crt --cacert ca.crt
```

Since this can be fairly painful to do, we can proxy the machines 8001 port to 6443 of the API server by using the `kubectl proxy` command. After this, we can easily run use the REST API to talk to the kube api. 



## Authorization

What can one do after authentication is called Authorization.  4 different types of authorization supported by k8s

- Node
- Attribute based authorization
- Role based authorization
- Webhook 

### Node 

The user talks to the API server to get status of nodes.

The kubelet running on every node reads and writes information to the API server. The crt for the kubelet belongs to the `system:node` group.

K8s has a special authorizer called the `Node Authorizer` that gives authorization tot he kubelet on the node to talk to the API server to do the following 

READ
- Services
- Endpoints
- Nodes
- Pods

WRITE
- Node status 
- Pod Status
- events


### Attribute baed Access Control

You associate a user or a group of users with a set of permissions. This is done by passing a user or a group to the API server in JSON format such as

```json
{
    "kind": "Policy",
    "spec": {
        "user": "dev-user1",
        "namespace": "*",
        "resource": "pods",
        "apiGroup": "*"
    }
}
{
    "kind": "Policy",
    "spec": {
        "group": "dev-users",
        "namespace": "*",
        "resource": "pods",
        "apiGroup": "*"
    }
}
```