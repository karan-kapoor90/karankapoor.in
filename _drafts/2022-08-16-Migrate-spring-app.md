# Reference Material
- Architectural guide - https://cloud.google.com/architecture/migrate-tomcat-to-containers-with-migrate-to-containers 

# Using CLoud Build and buildpacks

Buildpacks can autodetect the language and build container images for you without needing a cloudbuild.yaml or a dockerfile.

```bash
gcloud builds submit --pack image=gcr.io/[project-id]/my-app
```

# Sourcing the recipes 

<!---

1. Clone the repo at `https://github.com/spring-guides/gs-rest-service` into your local machine or cloud shell (we'll assume you're doing it in cloud shell).
2. Change directory into the project folder and then into the `complete` folder. To simply build your app into a jar file `./mvnw clean package` or you can run the app locally using the command `./mvnw spring-boot:run`.
3. Click on the Web Preview button on top of the cloud shell terminal section and in the new preview window that opens, change the path at the end of the URL to `/greeting`.
4. You'll notice that the app is running here. 

# Running the app on a VM


> This did not work

1. Create a gcp linux compute instance VM. Ensure you create the correct firewall rules to open ports 22 for ssh and 8080 to view the spring application. 
2. SSH into this VM, run `sudo apt install default-jdk git` to install OpenJDK and git.
3. Follow the same steps as in the section `Sourcing the recipes`, except in the end instead of using `./mvnw spring-boot:run`, use the command `./mvnw spring-boot:run &` so the app keeps running in the background.

> This worked
--->
Simply follow [this guide](https://digitalocean.com/community/tutorials/how-to-install-apache-tomcat-9-on-debian-10) to install and setup tomcat on the VM, just remember to install the [latest version](https://dlcdn.apache.org/tomcat/tomcat-10/v10.0.23/bin/apache-tomcat-10.0.23.tar.gz) instead of some old version.  

# Run the mFit assessment

1. SSH into the VM where the app is running and run the following:

```bash
mkdir mfit
cd mfit
```
2. Download and provide run permissions to the mfit tools that run the assessment. 

```bash
curl -O "https://mfit-release.storage.googleapis.com/$(curl -s https://mfit-release.storage.googleapis.com/latest)/mfit"
chmod +x mfit
```

If it's a single VM you want to do this with, you can download the linux data collection script, which will create a tar that can then be taken to a machine where mfit is available, and then mfit can discover by reading the tarball. [See - install mfit-linux-collection.sh](https://cloud.google.com/migrate/containers/docs/mfit-install) - this is what we will be doing since we're running it on a single VM that we've ssh'd into. 

```bash
curl -O "https://mfit-release.storage.googleapis.com/$(curl -s https://mfit-release.storage.googleapis.com/latest)/mfit-linux-collect.sh"
chmod +x mfit-linux-collect.sh
```

3. Run the dicovery inside the VM 

```bash
sudo ./mfit-linux-collect.sh
```

4. This creates a tar file that can then be discovered using the mfit tool

```bash
./mfit discover import /path/to/tar/file.tar
```

5. Check that the discovery has completed successfully

```bash
./mfit discover ls
```

# Assess and report the data discovered

1. We can create an html (directly readable), json (upload to gcp console), or csv (diy readable) report. For this demo, we'll create a json report that we'll upload into the GCP console to assess.

```bash
./mfit assess
./mfit report --format json > report-name.json
```

2. SSH out of your VM instance and scp the json report onto the cloud shell from where it can be uploaded to the tool. 

```bash
gcloud compute scp instance-1:~/mfit/myreport.json myreport.json --zone=us-central1-a
```

3. Navigate to the gcp UI > Migrate to Containers, and click on `Open Fit Assessment Report` and upload the json file here. Since the tomcat server is running and there's an app running inside it, you'll notice that it gets picked up as a valid candidate for Tomcat Migration.

# Setting up for migration

The processing cluster is a GKE cluster that has additional Migrate for Containers components installed, which are leveraged to convert the workloads detected earlier, to containers. So you've got a Kubernetes cluster converting conventional non container workloads into containers! How cool is that!

The type of processing cluster depends on the source - https://cloud.google.com/migrate/containers/docs/setting-up-overview#choose_the_type_of_processing_cluster

Some best practices for planning a migration here - https://cloud.google.com/migrate/containers/docs/planning-best-practices 

## Creating service account

m4c needs 2 service accounts - 
- 1 for accessing container registry and cloud storage - this is where the eventual container images will go.
- 1 for accessing compute engine as a migration source. 

I used the same service account for both - this serviceaccount has owner privileges on the porject, but clearly that's malpractice. Then I downloaded the service account key in json format to be used later.

## Setting up the cluster

```bash
gcloud container clusters create processing-clu \
 --project kay-dev-project \
 --zone=us-central1-a \
 --num-nodes 1 \
 --machine-type "e2-standard-4" \
 --image-type "UBUNTU" \
 --network anthos-sample-deployment-2-net \
 --subnetwork anthos-sample-deployment-2-subnet
```

then connect the cluster - this is only if you want to use the command line to do the installation etc. If you want to use the GCP Ui to perform the migration, this next step can be skipped.

```bash
gcloud container clusters get-credentials processing-clu --zone us-central1-a --project kay-dev-project
```

## Get catalina locations

In the VM where the tomcat server is running, navigate to the tomcat home directory (`/opt/tomcat` in this case) and run `./bin/catalina.sh version` to get the values for the Catalina base and the catalina home. We will be needing these values later.  

# Creating the migration

1. Navigate to Migrate for Containers and click `Add processing Cluster` and add the processing cluster and the service account we created earlier.
2. Navigate to add sources, and add a new source. Mention the processing cluster as the processing cluster, select a relevant name for the source, and select compute engine as the source type.
3. Select `Use existing Service Account` and paste the contents of the json key fole from the service account you downloaded earlier, and click add source.
4. Once done, navigate to the migrations tab and create a new migration. Give it a name, select the source we created earlier from the dropdown, choose the workload type as Tomcat Cntainer and give the name of the vm instance. This is where you mention the Catalina base and catalina home we picked up from the VM earlier.
5. Create the Migration. 
6. Once the migration is created, the mmigration plan is created (You can ideally modify the migration plan to add the default repository name where the images will be pushed, to avoid having to change it all over the place later). Click generate Artifacts to generate the actual artifacts.

# Applying the artifacts


`migctl migration get-artifacts [migratiojn-name]` - use this to get the artifacts generated by the migration.
`skaffold build -d us.gcr.io/[project-id]` - used to build the container images and push to us.gcr.io
`skaffold render --digest-source=remote -d us.gcr.io/[project-id]` - used to generate the single kubernetes yaml file which will have all the relevant k8s artifacts that need to be deployed to the target cluster. pa 

----
# running the app on tomcat

- add a `packaging` tag to pom.xml and use the value `war`
- in the build section add a tag called `<finalName>${project.artifactId}</finalName>` - to give the jar a non version dependent name.
- package using the command `./mvnw clean package -P production` [might be pretty useful since this code doesn't have a production profile]
- Install the tomcat server using this blog https://digitalocean.com/community/tutorials/how-to-install-apache-tomcat-9-on-debian-10
- tomcat version downloaded and installed - https://dlcdn.apache.org/tomcat/tomcat-10/v10.0.23/bin/apache-tomcat-10.0.23.tar.gz 


sudo -u admin_kaykapoor_altostrat_com