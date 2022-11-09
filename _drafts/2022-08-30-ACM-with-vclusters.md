# Installing vluster CLI

curl -s -L "https://github.com/loft-sh/vcluster/releases/latest" | sed -nE 's!.*"([^"]*vcluster-linux-amd64)".*!https://github.com\1!p' | xargs -n 1 curl -L -o vcluster && chmod +x vcluster;
sudo mv vcluster /usr/local/bin;

# Create a vcluster 

vcluster create mycluster --expose

# Connecting to a vcluster

vcluster connect mycluster --update-current=false

then simply change to the context for the vcluster, or source the kubeconfig variable with the kubeconfig file.

Connect the cluster to the gke hub fleet by creating a membership for the vcluster

References: https://cloud.google.com/anthos/clusters/docs/attached/how-to/attach-kubernetes-clusters#gcloud 

gcloud container fleet memberships register vcluster-mycluster \
   --context=vcluster_mycluster_vcluster-mycluster_c1 \
   --kubeconfig=./kubeconfig.yaml \
   --service-account-key-file=./vcluster-mycluster-sa-atlan-key.json

# Granting user access to a vcluster using their google cloud creds

Reference: https://cloud.google.com/anthos/multicluster-management/gateway/setup
Alternatively if you want to give access using Groups: https://cloud.google.com/anthos/multicluster-management/gateway/setup-groups 

Since my user is already a project owner, I don't need to define any iam-role-bindings, but since RBAC permissions still need to be given inside the cluster - I'm providing my user clusterrole/cluster-admin but you can create your own custom roles and cluster roles on the vcluster and provide them to users. This can help with more granular access control in the clusters.

gcloud beta container fleet memberships generate-gateway-rbac  \
    --membership=vcluster-mycluster \
    --role=clusterrole/cluster-admin \
    --users=admin@kaykapoor.altostrat.com \
    --project=kay-dev-project \
    --kubeconfig=./kubeconfig.yaml \
    --context=vcluster_mycluster_vcluster-mycluster_c1 \
    --apply

Simply refresh your google cloud console after this and you're good to go - now you can see the tenant vcluster on anthos UI as a registered cluster.

Now we move on to syncing with a repo






