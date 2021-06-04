## Kubelet

Its the component that talks tot he API server and does everything on the worker nodes wrt to creating containers etc. 

Kubelet runs as a service on the worker nodes. The config is passed as startup parameters tot he service. Those configs can also be passed in as a config file.

kubelet has 2 ports 
- 10250 - used to talk to the api-server. Read-Write
- 10255 - serves an api that has unauthenticated read-only access - insecure

10255 gives metrics endpoint for viewers. This is a security threat, while 10250 can also be used unauthenticated if a user knows the IP etc. of the worker nodes. 

This can be prevented via authentication and authorization. 

### Authentication:

To disable unautorized access to the 2 ports, in the kubelet config (the service or the configuration file), we can use the `-anonmous-auth=false` flag to disable access.

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
```

Supported authentication methods are Cert based auth and token based auth. 

*** Certificate Based: ***

pass the parameter `--client-ca-file=/path/to/ca.crt` or int he kubelet config file, the yaml file:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  x509:
    clientCAFile: /path/to/ca.crt
```

Now to auth into the kubelet while doing the curl, you'll have to pass the cert and the key

```bash
curl -sk https://localhost:10250/pods -key kubelet-key.pem -cert kubelet-cert.pem
```

For the kubelet, the kube-apiserver is a client. Which means that the kube-apiserver also needs client certs to auth with the kubelet. For this the apiserver must have the kubelet client cert and key in the startup params. `--kubelet-client-certificate` and `--kubelet-client-key`


### Authorization

Default authorization mode is `AlwaysAllow`. This is set in the kubelet service configs `--authorization-mode=AlwaysAllow` and also in the yaml.

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  mode: AlwaysAllow
```


When the mode is changed to `Webhook` the kubelet sends a request to the apiserver to ascertain if each one of the requests can be completed or not. For reconfiguring the kubelet config service use the flag `--authorization-mode=Webhook` or in the yaml file.

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authorization:
  mode: Webhook
readOnlyPort: 0
```

> By default port 10255 is disabled. This happens by either an absent `--read-only` parameter in the kubelet config, or a setting of `readOnlyPort: 0` in the kubelet config. 


