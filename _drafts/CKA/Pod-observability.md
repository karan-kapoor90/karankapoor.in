## Readiness Probes

Even when the pod status is Ready, it can still take some time for the actual service to come up. Wether or not a container is ready (from an application up and running perspective), can be determined using readiness probes in the pod definition. There can be:

- an API endpoint exposed by the app running in the container
- a TCP socket that becomes active in the container
- an exec command that can run a shell command (custom defined) inside the container to check if the container is ready

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    name: my-pod
spec:
  containers:
  - name: my-pod
    image: nginx
    ports:
      - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /api/ready
        port: 8080
      initialDelaySeconds: 60    # Delay first check by 60 seconds
      periodSeconds: 5   # How often to probe. Check after every 5 seconds
      failureThreshold: 8   # Normally probe fails after 3 failed attemps. This increases number of attempts
```

The other options are:

```yaml
# httpGet
readinessProbe:
  httpGet:
    path: <ready url path>
    port: <service port>

# tcp socket
readinessProbe:
  tcpSocket:
    port: 3036

# exec
readinessProbe:
  exec:
    command:
    - cat
    - app/is/ready
```

## Liveness Probes

What if the application is stuck in an infinite loop or has some code error. k8s would assume that since the container is up and running, all is well, and would hence not recreate the pod. This can be be prevented using Liveness Probes. LP will check if the app in the code is healthy

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-pod
spec:
  containers:
  - name: my-pod
    image: nginx
    livenessProbe:
      httpGet:
        path: api/health
        port: 8080
      initialDelaySeconds: 60
      periodSeconds: 5
      failureThreshold: 8
```

This too, like readiness probes allows for:
- httpGet
- tcpSocket
- exec

