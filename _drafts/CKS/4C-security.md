# 4C's of Security

1. Cloud
2. Cluster
3. Containers
4. Code

# CIS benchmark

On VM/ physical hosts, root user should be disabled and all users should have their own usernames. Admin users should be allowed sudo permissions for performing tasks needing root permissions. 

Firewalls and IPtables should be setup

Services that are not required should be disabled
File system access should be restricted
Auditing should be setup to log what users are doing. 


CIS - Center for Internet Security

CIS-CAT tools help run the benchmark and create reports showing compliance.

To run a vulnerability assessment on the Linux machine, use the CSI-CAT Pro Assessment tool. 

```bash
sh Assessment-CLI.sh -i -rd /var/www/html/ -nts -rp index
```

CIS-CAT Pro is paid, and the lite version is not available for k8s. An OSS alternative is Acqua Security kube-bench. 

1.3.6 Edit the Controller Manager pod specification file /etc/kubernetes/manifests/kube-controller-manager.yaml
on the master node and set the --feature-gates parameter to include RotateKubeletServerCertificate=true.
--feature-gates=RotateKubeletServerCertificate=true


# Security Primitives

Controlling access to the API Server
- Who can access the API server - authentication
    - Files - Username and Password
    - Files - Usernames and tokens
    - Certificates
    - External Auth providers - LDAP
    - Service Accounts
- What can they do - authorization
    - RBAC Authorization
    - ABAC Authorization
    - Node Authorization
    - Webhook Mode