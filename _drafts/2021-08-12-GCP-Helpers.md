#Installation tips and tricks

kaykapoor@



## Creating GKE clusters with Istio installed

```bash
gcloud beta container clusters create $CLUSTER_NAME \
    --zone $CLUSTER_ZONE --num-nodes 4 \
    --machine-type "n1-standard-2" --image-type "COS" \
    --cluster-version=$CLUSTER_VERSION --enable-ip-alias \
    --addons=Istio --istio-config=auth=MTLS_STRICT
```


### Installing Istioctl

```bash
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=latest sh -
cd ./istio-*
export PATH=$PWD/bin:$PATH
```



## Creating clusters with particular features enabled

### Enabling Stack Driver 

```bash
export PROJECT_ID=$(gcloud config get-value project)
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_ID}"

gcloud beta container clusters create ..... --enable-stackdriver-kubernetes --workload-pool=${WORKLOAD_POOL} --labels mesh_id=${MESH_ID}
```

### Enabling access to Cloud Source Repo

```bash
gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"   # the scope section is what enables RW access to Cloud Source Repo.
```


## Application Specific

### Create a Cloud Source Repo - Git

```bash
# Note: You will need to change the <repo-name> throughout

gcloud source repos create <repo-name>
gcloud source repos clone <repo-name>

git init
git config credential.helper gcloud.sh
git remote add origin https://source.developers.google.com/p/$DEVSHELL_PROJECT_ID/r/<repo-name>
git config --global user.email "[EMAIL_ADDRESS]"
git config --global user.name "[USERNAME]"
```


### Generating steady background load

```bash
sudo apt install siege
siege http://${GATEWAY_URL}/productpage
```

### Generate steady load and metrics - hey_linux

```bash

# download `hey`
curl -o hey_linux_amd64 \
https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
chmod +x hey_linux_amd64
# -z 5m == run for 5 minutes
# -q 50 == 50 QPS per worker 
# -c 10 == 10 concurrent workers
./hey_linux_amd64 -z 5m -q 50 -c 10 https://<url-to-load>
```

### Generate load using apache2-utils

```bash
sudo apt update
sudo apt install apache2-utils -y
ab -n 1000 -c 10 http://<>  // Will make 1000 requests 10 at a time to your app
```


### Installing Jenkins on GKE

Reference : https://www.qwiklabs.com/focuses/1103?parent=catalog 

```bash
git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
cd continuous-deployment-on-kubernetes
helm add repo jenkins https://charts.jenkins.io
helm repo update
helm install cd jenkins/jenkins -f jenkins/values.yaml --version 1.2.2 --wait
```

> values.yaml from the repo cloned above has soom good default packages and setting to get started with Jenkins.


## Quick Demo Assets

### Demo Logging Tracing and Monitoring

This will deploy a python app with cloud profiler enabled

```bash
git clone https://GitHub.com/GoogleCloudPlatform/training-data-analyst.git
cd training-data-analyst/courses/design-process/deploying-apps-to-gcp
```

In main.py add `import googlecloudprofiler` on line 2 and then the following block below the main function

```python
try:
    googlecloudprofiler.start(verbose=3)
except (ValueError, NotImplementedError) as exc:
    print(exc)
```

Add to requirements.txt `google-cloud-profiler` and create a file called app.yaml and add the text `runtime: python37`

```bash
// To run locally

gcloud services enable cloudprofiler.googleapis.com
sudo pip3 install -r requirements.txt
python3 main.py

// To deploy to app engine

gcloud app create --region=us-central
gcloud app deploy --version=one --quiet
```

Put some load on the application. Get the app URL from the App Engine Page

```bash
// Create a VM and install apache utils
sudo apt update
sudo apt install apache2-utils
ab -n 1000 -c 10 <GAE_app_url>
```

Then navigate to cloud profiler, trace and debugger to see all that the app generated. You cam also use the URL on the app to create an uptime check and an alert to email from cloud monitoring 

### Interesting images:

Returns the name of the app server as a response

```yaml
spec:
      containers:
      - image: k8s.gcr.io/serve_hostname:v1.4
        name: hostname-server
        ports:
        - containerPort: 9376
          protocol: TCP
      terminationGracePeriodSeconds: 90
```

App that says hello v1

```yaml
spec:
      containers:
      - name: hello-app-1
        image: "us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0"
        env:
        - name: "PORT"
          value: "50000"
```

App that says hello v2

```yaml
spec:
      containers:
      - name: hello-app-2
        image: "us-docker.pkg.dev/google-samples/containers/gke/hello-app:2.0"
        env:
        - name: "PORT"
          value: "8080"
``` 

## Build and deploy automation

### skaffold config for cloud build and push only

This skaffold yaml, when executed locally using the following command, will build containers using cloud build and then push them to GAR using this config and the ensuing command.

```yaml
apiVersion: skaffold/v2beta7
kind: Config
build:
  artifacts:
    - image: leeroy-web
      context: leeroy-web
    - image: leeroy-app
      context: leeroy-app
  googleCloudBuild:
    projectId: qwiklabs-gcp-02-b710f3acf6cf
deploy:
  kubectl:
    manifests:
      - leeroy-web/kubernetes/*
      - leeroy-app/kubernetes/*
portForward:
  - resourceType: deployment
    resourceName: leeroy-web
    port: 8080
    localPort: 9000
```

```bash
skaffold build --interactive=false --default-repo=$REGION-docker.pkg.dev/$PROJECT_ID/web-app --file-output artifacts.json
```

Check images in GAR using:

```bash
gcloud artifacts docker images list $REGION-docker.pkg.dev/$PROJECT_ID/web-app --include-tags --format=yaml
```

## Build images using pack without a Dockerfile

Run this in the project root such as the nodejs app folder where the package.json file is present.

```bash
gcloud builds submit --pack image=us-central1-docker.pkg.dev/$PROJECT/helloworld-repo/helloworld
```