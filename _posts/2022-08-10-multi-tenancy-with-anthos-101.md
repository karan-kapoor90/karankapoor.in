---
layout: post
categories: [Tutorials, Multi-tenancy]
comments: true
excerpt: A 101 to Anthos for multi-tenancy using Anthos Config Management
title: Multi-tenancy 101 using Google Anthos Conifg Management
tags: [kubernetes, GCP, Anthos, Git-Ops]
description: A guide to setup a basic environment for Git-ops based multi-tenancy using Anthos Config Management
image:
  feature: acm.png
---


# Background
 
I'm a platform admin tasked with running a multi-tenant application on GKE. This leaves me with a couple of fundamental things to do:
- I need to create the GKE cluster where every tenant's app will be deployed.
- I need to create the policies, namespaces etc. that will govern every tenant. For example: every tenant gets their own GKE namespace in a large shared cluster
- I should be able to put all my platform level GKE configurations into a git repository and keep them in sync with cluster
- For every tenant, I will create a new git repository per tenant - this will be used to hold the application code for every individual tenant, the namespace in the GKE cluster where every individual tenant's apps will be deployed, the synchronization b/w this tenant git repository, and the tenant namespace on the GKE cluster.  

> Any automation to setup the various components can be done using several pieces of technology like terraform, KRM, Crossplane, Cloud Build etc., but for the purpose of keeping this fairly high level, we'll stick to the basics and not use any of those automation tools at the moment.

# What are we going to be doing

## Terminology

<b>Admin:</b> This is the person responsible for the platform as a whole. This is the person who creates policies and resources that are common across the kubernetes cluster, and apply to all tenants. This admin is NOT responsible for deploying the code that is used by every individual tenant.

<b>App Owner:</b> This is the person responsible for the particular tenant. This person deploys an instance of the application that will be used by the particular tenant.

<b>Root Repo:</b> This is a git repo where the components set up by the Admin are kept. App Owners do not have access to this.

<b>Namespace Repo:</b> This is a git repo for a particular tenant, a different git repo for every tenant. So let's say you have 2 customers A, and B. Both of these will be 2 individual tenants, and will have their own respective git repos called Namespace Repos.

## Construct

The admin creates the cluster where every tenant's code will live. All of the Admin's configuration will go into one single git repository, the root repository, and will reside inside one single namespace called `admin-ns`. On receiving a request to onboard a new tenant to the cluster, the admin creates a new folder in their git repository (the root repo) for the tenant, and puts in a namespace yaml file, and a role binding for that tenant in that namespace folder. This ensures that the namespace and the role binding for the individual tenant are always synced in the cluster, and any tenant cannot delete it. 

The admin then creates a new repository for the tenant called the namespace repo, and adds the configuration to sync this repo, with the namespace created for the particular tenant. This syncing is again controlled by the admin, and nobody should have the permission to remove this sync. Hence these components must also go into the root repo, in the particular namespace folder, maintained by the admin. 

The admin then gives access to this new repo, the namespace repo, to the App Owner. The app owner then adds the tenant specific code, into this namespace repo so that the app resources deployed in the k8s cluster for this tenant are always in sync with the resources in the namespace repo. 

There are other ways to structure this as well, which are explained and demonstrated in the [official Anthos documentation here](https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories). 

# Setting up your environment

This guide will use a bunch of env. variables across the yaml and commands to help simplify the whole process. Let's set them up here.

```bash
export PROJECT_ID=$(gcloud config get project)
export CLUSTER_NAME=my-cluster
export COMPUTE_REGION=us-central1
export NAMESPACE=admin-ns
export KSA_NAME=root-reconciler
export GSA_NAME=my-gsa
export TENANT_NAME=tenant-1
```
 
# Creating GKE cluster
 
Leveraging workload identity on the GKE cluster enables the comms b/w the Git repo on GCP using a service account instead of tokens or ssh keys and is hence the recommended way of doing it. Basically, instead of creating a secret on the GKE level and using that to enable connectivity b/w the cluster worker nodes and the Source Repo, the service account on the worker node gives the worker node, service account level access to allow the node to talk to the source repo.
 
- Create a regular public GKE cluster but ideally with Workload identity enabled on the GKE cluster and the GKE Metadata server on the worker node pool.
    ```bash
    gcloud container clusters create $CLUSTER_NAME \
    --region=$COMPUTE_REGION \
    --workload-pool=$PROJECT_ID.svc.id.goog
    ```
- If the cluster already exists, enable Workload identity on the cluster level and the GKE metadata server on the nodepools.
    ```bash
    gcloud container clusters update $CLUSTER_NAME \
    --region=$COMPUTE_REGION \
    --workload-pool=$PROJECT_ID.svc.id.goog
    ```

In your cloud shell (we're assuming you're using it for the purpose of this demo), get access to the kubernetes cluster you just created using the following command:

```bash
gcloud containers cluster get-credentials $CLUSTER_NAME --region=$COMPUTE_REGION
```

# Ready Cloud Shell to work with the source repo
 
1. Create a repo on Cloud Source Repo from the GCP UI.
2. Generate an ssh key on the cloud shell environment to be able to push and pull from this repo we just created. 
 
   ```bash
   ssh-keygen -t rsa
   cat ~/.ssh/id_rsa.pub
   ```
3. On the cloud source repo page, click on the 3 dots on the top right on the page, click on Manage SSL keys and create a new key, give it a name and paste the ssh key contents from the previous step there.
 
4. Clone the repository using the ssh url to the cloud shell.
5. Add your code to the repo folder
 
   ```bash
   mkdir home
   touch home/README.md
   ```
 
6. git add, git commit, add remote and then push the code to the main branch to the cloud source repo.
 
We will now use this cloud source repo as the repo to sync with the kubernetes cluster using config sync.
 
# Service account setup

> This is the service account mentioned in the [Creating GKE Cluster](#creating-gke-cluster) section. 

1. Create a service account in your GCP project using the following command, and give it the appropriate roles on GCP for the job. 
    ```bash
    gcloud iam service-accounts create $GSA_NAME --project=$PROJECT_ID
    gcloud projects add-iam-policy-binding $PROJECT_ID --member "serviceAccount:$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com" --role roles/source.writer
    ```

    Normally at this point you'd create a service account inside your kubernetes cluster (called KSA), and annotate it with the Google service account (GSA) we just created above. In this case however, since we're going to be using Anthos Config Management (ACM), it auto-creates the Service account inside the cluster that it uses. So we simply need to annotate this service account created by ACM, with the name of the GSA. We will be doing it in [Step 10](#configure-acm-into-the-cluster) in the next section.

    Note: The service account we've created on the GCP side in the previous step is called the `GSA` = `Google Service Account`. The Service account that'll be created in the kubernetes cluster is consequently going to be called the `KSA` = `Kubernetes Service Account`. Get the analogy?
 
# Configure ACM into the cluster
 
1. On the Anthos UI, navigate to clusters and select the gray box that asks you to register the un-registered cluster. There, select the cluster we created earlier (which is currently not enrolled in Anthos), and enroll it by clicking on the Register button next to the name of the cluster.

    ![Onboarding cluster to Anthos](/img/cluster_onboarding.gif "Onboarding cluster to Anthos")

2. Navigate to the config management tab on the left hand side and use the gray box on the top to set up config management for the cluster where it hasn't been set up. Setup steps as follows.
    ![Enabling Config Sync inside cluster](/img/acm_onboarding.gif "Enabling Config Sync inside cluster")

3. Select the cluster on the next screen and hit continue.
4. You can unselect Policy Controller for now and hit continue
5. Keep the Enable config sync checkbox selected and select custom from the dropdown menu.
6. In the URL field, put the HTTPS protocol URL for the repo in the Cloud Source Repo. Do not put the ssh url since we're using Service account as the mode of authentication, and not ssh keys.
7. Click on Advanced Settings and select Authentication type as Workload identity. In the service account email field, mention the service account email ID from the previous step.
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

[Scenario 1](https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories#manage-namespace-repos-in-root): The admin creates the basic artifacts such as the tenant namespace in the cluster, the synchronization artifacts of the tenant specific git repo, service account role bindings etc.

[Scenario 2](https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories#manage-namespace-repos-in-namespace): you can simply let that be managed by the cluster tenant owner (the app owners or whoever is responsible for deploying the app in the cluster namespace). So the app owner will have to set up the sync, the service account mappings etc.
 
In this document, we're covering Scenario 1.
 
We now intend to add a tenant to the cluster and the git repository to be synced for this tenant is referred to as the namespace repository. The k8s yaml for creating the namespace, relevant admin level artifacts such as network policies, RepoSync configuration for a tenant repository, stay in the root repository, while things such as the deployments, services, ingress etc resources that are specific to a tenant may go in the namespace repo.
 
## Creating a namespace in the cluster for the tenant
 
> This section is performed by an admin, and not the application owners.
 
1. Add a yaml for creation of the tenant namespace to the root repo in the `home/` folder.
 
    ```bash
    cat <<EOF > home/$TENANT_NAME/ns-$TENANT_NAME.yaml 
    apiVersion: v1
    kind: Namespace
    metadata:
     creationTimestamp: null
     name: $TENANT_NAME
    spec: {}
    EOF
    ```
 
## Creating a git repo for the tenant
 
> This section is performed by an admin, not an app owner.
 
1. Create a git repo for the tenant from the UI, or using the gcloud CLI as desired.
 
> Make note of the https repo URL for the repo that's been created. We'll need it in the next steps to setup the sync. Also, we'll assume here that the app owners will put the kubernetes manifests in a `manifests` folder in the root of the repo.
 
## Setting up sync b/w tenant namespace & tenant git repo
 
> This section is performed by an admin, not an app owner.
 
1. The admin now needs to create a repo sync resource that will sync the resources created in a tenant git repo `tenant-1` namespace we created - this repo sync resource is a namespaced resource and needs to be applied into tenant-1's namespace by the admin.
 
    ```bash
    cat <<EOF > home/$TENANT_NAME/reposync.yaml
    apiVersion: configsync.gke.io/v1alpha1
    kind: RepoSync
    metadata:
     # There can be at most one RepoSync object per namespace.
     # To enforce this restriction, the object name must be repo-sync.
     name: $TENANT_NAME-reposync
     namespace: $TENANT_NAME
    spec:
     # Since this is for a namespace repository, the format should be unstructured
     sourceFormat: unstructured
     git:
       # If you fork this repo, change the url to point to your fork
       repo: https://source.developers.google.com/p/$PROJECT_ID/r/$TENANT_NAME
       # If you move the configs to a different branch, update the branch here
       branch: main
       dir: manifests/
       gcpServiceAccountEmail: $GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com
       # We recommend securing your source repository.
       # Other supported auth: `ssh`, `cookiefile`, `token`, `gcenode`.
       auth: gcpserviceaccount
    EOF
    ```
 
2. Kubernetes Service accounts or KSA's are namespaced resources and hence the `tenant-1` namespace also needs it's service account to be linked to a GSA (a Google Service Account) so the `RepoSync` object has the permissions to pull the artifacts from the source repository. Note, that this KSA which needs to be attached with the GSA is only created after the RepoSync object has been applied to the cluster.
 
    Note: In the case of a `root-repository` sync, the KSA name is `root-reconciler`, while in the case of a namespace repo, the KSA starts with the prefix `ns-reconciler` and could have a name like `ns-reconciler-[namespace]` or `ns-reconciler-[repo_sync_name]`. This can be checked using the command `kubectl get sa -A | grep ns-reconciler`
 
3. Once we have created the RepoSync object, proceed to attach the KSA to the GSA using the following command:
 
    ```bash
    export RECONCILER=$(kubectl get sa -n config-management-system -o=custom-columns=name:.metadata.name | grep ns-reconciler)
    gcloud iam service-accounts add-iam-policy-binding \
      --role roles/iam.workloadIdentityUser \
      --member "serviceAccount:$PROJECT_ID.svc.id.goog[config-management-system/$RECONCILER]" \
      $GSA@$PROJECT_ID.iam.gserviceaccount.com
    ```
 
4. The KSA above will need the appropriate permissions within the cluster to CRUD resources in the cluster, such as managing deployments, services etc. This service account can be allocated a custom or a pre-defined Kubernetes Role or a ClusterRole (such as the omnipotent `admin` - clearly a bad idea for production). In the root repo, create this yaml file to be synced as well.
 
    ```bash
    cat <<EOF > home/$TENANT_NAME/rolebinding-syncs-repo.yaml
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
     name: syncs-repo
     namespace: $TENANT_NAME
    subjects:
    - kind: ServiceAccount
      name: $RECONCILER
      namespace: config-management-system
    roleRef:
     kind: ClusterRole
     name: admin
     apiGroup: rbac.authorization.k8s.io
    EOF
    ```
 
5. Check the status of the sync using the following command
 
    ```bash
    nomos status
    ```
 
# Add deployment artifacts to the tenant Git repo
 
> This section is performed by an App owner, not an admin.
 
1. Clone the tenant git repository that was created and supplied by the admin, into the cloud shell.
2. Add the following simple nginx deployment yaml to the repository in a new folder (that you create) called `manifests`. Make sure there is no `status` field in the deployment spec.
 
    ```bash
    cat <<EOF > manifests/deployment-nginx.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
     creationTimestamp: null
     labels:
       app: nginx
     name: nginx
     namespace: $TENANT_NAME
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
    EOF
    ```
 
3. Create a yaml file to expose the nginx deployment using a Load balancer in the same `manifests` folder. Make sure there is no `status` field in the service spec.
 
    ```bash
    cat <<EOF > manifests/service-nginx.yaml
    apiVersion: v1
    kind: Service
    metadata:
     creationTimestamp: null
     labels:
       app: nginx
     name: nginx
     namespace: $TENANT_NAME
    spec:
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app: nginx
    EOF
    ```
 
4. Add these changes, commit and push them to the repository.
 
5. You should now be able to view the nginx deployment and the accompanying load balancer service deployed to the `tenant-1` namespace in the cluster.
 
# References
 
    Doc Reference: https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync
    Multi-repo sync: https://cloud.google.com/anthos-config-management/docs/config-sync-quickstart
    Multi-repo documentation: https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories
    Repo types in ACM: https://cloud.google.com/anthos-config-management/docs/config-sync-overview#repositories
    Root-sync resource ref documentation: https://cloud.google.com/anthos-config-management/docs/reference/rootsync-reposync-fields#configuring-source-type 
