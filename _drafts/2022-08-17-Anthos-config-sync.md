# Background

I'm a platform admin tasked with running a multi-tenant application on GKE. This leaves me with a couple of fundamental things to do:
- I need to create the GKE cluster where every tenant's app will be deployed.
- I need to create the policies, namespaces etc. that will govern every tenant. For example: every tenant gets their own GKE namespace in a large shared cluster
- I should be able to put all my platform level GKE configurations into a git repository and keep them in sync with cluster
- For every tenant, I will create a new git repository per tenant - this will be used to hold the application code for every individual tenant, the namespace in the GKE cluster where every individual tenant's apps will be deployed, the synchronization b/w this tenant git repository, and the tenant namespace on the GKE cluster.   

# Creating GKE cluster

Leveraging workload identity on the GKE cluster enables the comms b/w the Git repo on GCP using a service account instead of tokens or ssh keys and is hence the recommended way of doing it. Basically, instead o creating a secret on the GKE level and using that to enable connectivity b/w the cluster worker nodes and the Source Repo, the service account on the worker node enables gives the worker node service account level access to allow the node to talk to the source repo.

- Create a regular public GKE cluster but ideally with Workload identity enabled on the GKE cluster and the GKE Metadat server on the worker node pool.
- If the cluster already exisits, enable Workload identity on the cluster level and the GKE metadata server on the nodepools.

# Ready Cloud Shell to work with the source repo

1. Create a repo on Cloud Source Repo
2. Generate an ssh key on the cloud shell environment

    ```bash
    ssh-keygen -t rsa
    cat ~/.ssh/id_rsa.pub
    ```
3. On the cloud source repo page, click on the 3 dots on the top right on the page, click on Manage SSL keys and create a new key, give it a name and paste the ssh key contents from the previous step there. 

4. Clone the repository using the ssh url to the cloud shell.
5. Add your code to the repo folder

    ```bash
    mkdir home
    kubectl create ns something --dry-run=client -o yaml > home/my-ns.yaml
    ```

6. git add, git commit, add remote and then push the code to the main branch to the cloud source repo. 

We will now use this cloud source repo as the repo to sync with the kubernetes cluster using config sync. 

# Service account setup

1. navigate to service accounts on the IAM and Admin Page and create a new service account.
2. Navigate to the IAM tab on the left hand side and select the service account email ID you just created. Grant it the least privilege `source.reader` role or the `source.writer` role for a wider range of permissions. 
3. Note the email ID for the service account on the service accounts page or the IAM page. We will need it later to setup service account matching b/w gcp service account and kubernetes service account. Note: The kubernetes service account from ACM gets created only after ACM has been installed on the cluster, hence this step is pending till the time we finish the next step of installing ACM into the cluster. 

Note: The service account we've created on the GCP side in the previous step is called the `GSA` = `Google Service Account`. The Service account that'll be created in the kubernetes cluster is consequently going to be called the `KSA` = `Kubernetes Service Account`. Get the analogy?

# Configure ACM into the cluster

1. On the Anthos UI, navigate to clusters and select the grey box that asks you to register the un-registered cluster. There, select the cluster we created earlier (which is currently not enrolled in to Anthos), and enroll it by clicking on the Register button next to the name of the cluster. 
2. Navigate to the config management tab on the left hand side and use the grey box on the top to setup config management for the cluster where it hasn't been setup. Setup steps as follows.
3. Select the cluster on the next screen and hit continue.
4. You can unselect Policy Controller for now and hit continue
5. Keep the Enable config sync checkbox selected and select custom from the dropdown menu.
6. In the URL field, put the HTTPS protocol URL for the repo in the Cloud Source Repo. Do not put the ssh url since we're using Service account as the mode of authentication, and not ssh keys. 
7. Click on Advanced Settings and select Authnetication type as Workload identity. In the service account email field, mention the service account email ID from the previous step. 
8. Set configuration directory to `home/`
9. Select the latest version of ACM and click Complete at the bottom to start the installation.
10. The Ui will show an error right now, and that's ok because the service account created inside GKE isn't connected to the GCP service account we created earlier, and hence the Config Sync component in the cluster doesn't have the required permissions to read the source git repo. 

Note: As per the [document](https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync#workload-identity-enabled), since the `root-reconciler` deployment is present in the `config-management-system` namespace, the name of the KSA is `root-reconciler`. Run `kubectl get sa -n config-management-system | grep root` to verify.   

    ```bash
    gcloud iam service-accounts add-iam-policy-binding \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[config-management-system/$KSA]" \
    $GSA@$PROJECT_ID.iam.gserviceaccount.com
    ```
11. With this, the root repository should sync with the home directory in the git repository. To check the status of the repo sync you can use the following command

    ```bash
    nomos status
    ```

# Adding another namespace repo, a tenant to sync

There are 2 models of operations here. 
[Scenario 1](https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories#manage-namespace-repos-in-root): The admin create sthe basic artifacts such as the tenant namespace in the cluster, the synchronization artifacts of the tenant specifc git repo, service account role bindings etc.
[Scenario 2](https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories#manage-namespace-repos-in-namespace): you can simply let that be managed by the cluster tenant owner (the app owners or whoever is reponsible for deploying the app in the cluster namespace). So the app owner will have to setup the sync, the service account mappings etc. 

In this document, we're covering Scenario 1.

We now intend to add a tenant to the cluster and the git repository to be synced for this tenant is refered to as the namespace repository. The k8s yaml for creating the namespace, relevant admin level artifacts such as network policies, RepoSync configuration for a tenant repository, stay in the root repository, while things such as the deployments, services, ingress etc resources that are specific to a tenant may go in the namespace repo.

## Creating a namespace in the cluster for the tenant

> This section is performed by an admin, and not the application owners.

1. Add a yaml for creation of the tenant namespace to the root repo in the `home/` folder.

```yaml 
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: tenant-1
spec: {}
```

## Creating a git repo for the tenant

> This section is performed by an admin, not an app owner. 

1. Create a git repo for the tenant from the UI, or using the gcloud CLI as desired.

> Make note of the https repo URL for the repo that's been created. We'll need it in the next steps to setup the sync. Also, we'll assume here that the app owners will put the kubernetes manifests in a `manifests` folder in the root of the repo.

## Setting up sync b/w tenant namespace & tenant git repo

> This section is performed by an admin, not an app owner. 

1. The admin now needs to create a repo sync resource that will sync the resources created in a tenant git repo `tenant-1` namespace we created - this repo sync resource is a namespaced resource and needs to be applied into tenant-1's namespace by the admin.

```yaml
apiVersion: configsync.gke.io/v1alpha1
kind: RepoSync
metadata:
  # There can be at most one RepoSync object per namespace.
  # To enforce this restriction, the object name must be repo-sync.
  name: tenant-1-reposync
  namespace: tenant-1
spec:
  # Since this is for a namespace repository, the format should be unstructured
  sourceFormat: unstructured
  git:
    # If you fork this repo, change the url to point to your fork
    repo: https://source.developers.google.com/p/kay-dev-project/r/tenant-1
    # If you move the configs to a different branch, update the branch here
    branch: main
    dir: manifests/
    gcpServiceAccountEmail: guestbook@kay-dev-project.iam.gserviceaccount.com
    # We recommend securing your source repository.
    # Other supported auth: `ssh`, `cookiefile`, `token`, `gcenode`.
    auth: gcpserviceaccount
```

3. Kubernetes Service accounts or KSA's are namespaced resources and hence the `tenant-1` namespace also needs it's service accocunt to be linked to a GSA (a Google Service Account) so the `RepoSync` object has the permissions to pull the artifacts from the source repository. Note, that this KSA which needs to be attached with the GSA is only created after the RepoSync object has been applied to the cluster. 

Note: In the case of a `root-repository` sync, the KSA name is `root-reconciler`, while in the case of a namespace repo, the KSA starts with the prefix `ns-reconciler` and could have a name like `ns-reconciler-[namespace]` or `ns-reconciler-[repo_sync_name]`. This can be checked using the command `kubectl get sa -A | grep ns-reconciler` 

4. Once we have created the RepoSync object, proceed to attach the KSA to the GSA using the following command:

```bash
gcloud iam service-accounts add-iam-policy-binding \
   --role roles/iam.workloadIdentityUser \
   --member "serviceAccount:kay-dev-project.svc.id.goog[config-management-system/ns-reconciler-tenant-1-tenant-1-reposync-17]" \
   guestbook@kay-dev-project.iam.gserviceaccount.com
```

6. The KSA above will need the appropriate permissions within the cluster to CRUD resoruces in the cluster, such as managing deployments, services etc. This service account can be allocated a custom or a pre-defined Kubernetes Role or a ClusterRole (such as the omnipotent `admin` - clearly a bad idea for production). In the root repo, create this yaml file to be synced as well.

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: syncs-repo
  namespace: tenant-1
subjects:
- kind: ServiceAccount
  name: ns-reconciler-tenant-1-tenant-1-reposync-17
  namespace: config-management-system
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

5. Check the status of the sync using the following command

```bash
nomos status
```

# Add deployment artifacts to the tenant Git repo

> This section is performed by an App owner, not an admin. 

1. Clone the tenant git repository that was created and supplied by the admin, into the cloud shell.
2. Add the following simple nginx deployment yaml to the repository in a new folder (that you create) called `manifests`. Make sure there is no `status` field in the deployment spec.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: tenant-1
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
        ports:
        - containerPort: 80
        resources: {}
```

3. Create a yaml file to expose the nginx deployment using a Load balancer int he same `manifests` folder. Make sure there is no `status` field in the service spec.

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: tenant-1
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
```

4. Add these changes, commit and push them to the repository. 

5. You should now be able to view the nginx deployment and the accompanying load balancer service deployed to the `tenant-1` namespace in the cluster.

# Setting up for multitenancy

Basis this [excellent guide](https://learnk8s.io/templating-yaml-with-code), I decided to atleast start with kustomize, and try to create a system for a multi-tenant deployment model using ACM. The eventual idea is to use a combination of Cloud Build, and ACM to create a multi-tenant setup which does the following:
- on calling a Cloud Build URL (relevant parameters such as the tenant's name are passed in as inputs), the following steps are executed: 
    - the gcloud command is used to create a cloud source repo for the tenant, and the URL for the repo captured. This will be the namespace repo which will carry the namespaced resources to be deployed for the individual tenant.
    - the root repo is checked out and a new tenant folder is setup in it, with all the relevant patch yamls, like the namespace, rolebinding and the reposync resources with the name of the tenant. The values are still be the original values copied over from the template but we use sed or yq to overwrite the values based on the params passed into the build pipeline. 
    - once modified, a new git commit will be created which will be pushed to the repo, which will then be synced with the main cluster.

# Autoscaler exception

This doesn't seem to play nice with HPA. Irrespective of whether you use HPA imperatively or decleratively, there seems to be contention in scaling due to which additional pods aren't created. 

```bash
kubectl run -i --tty --rm loadgen --image=cyrilbkr/httperf --restart=Never -- /bin/sh -c 'httperf --server=[target-IP] --hog --uri="/zone" --port 80  --wsess=100000,1,1 --rate 1000'
```


# References

Doc Reference: https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync 
Multi-repo sync: https://cloud.google.com/anthos-config-management/docs/config-sync-quickstart 
Multi-repo documentation: https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories
Repo types in ACM: https://cloud.google.com/anthos-config-management/docs/config-sync-overview#repositories
Root-sync resource ref documentation: https://cloud.google.com/anthos-config-management/docs/reference/rootsync-reposync-fields#configuring-source-type
Using templating engines: https://learnk8s.io/templating-yaml-with-code 
Use kustomize and Helm with ACM: https://cloud.google.com/anthos-config-management/docs/how-to/use-repo-kustomize-helm
Enable Drift prevention: https://cloud.google.com/anthos-config-management/docs/how-to/prevent-config-drift#kubectl 
Load Testing: https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways#verify_traffic_using_load_testing
Configuring HPA: https://cloud.google.com/kubernetes-engine/docs/how-to/horizontal-pod-autoscaling#kubectl-apply 
Official HPA docs: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ 
 