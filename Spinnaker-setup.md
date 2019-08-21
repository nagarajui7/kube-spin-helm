# Steps for installing Spinnaker
1. Install Halyard:

`curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh`

run it as ubuntu user

`sudo bash InstallHalyard.sh`

check the version using: `hal -v`

2.Setting up Docker-Registry Cloud Provider

`hal config provider docker-registry enable`

`hal config provider docker-registry account add my-docker --address index.docker.io --repositories 	library/nginx --username nagarajubatchu1 --password`

3. Setting up Kubernetes Cloud Provider
**Note**: Before enabling kubernetes provider, make sure .kube directory is present along with config file in it.

`hal config provider kubernetes enable`

`hal config provider kubernetes account add my-k8s --repositories my-docker`

`CONTEXT=kubectl config current-context`

`hal config provider kubernetes account add my-k8s-3 --provider-version v2 --context $CONTEXT`

4. Environment Type

`hal config deploy edit --type localdebian`

5. Setup Storage GCS

We will be creating an ***iam role*** and a auth token for spinning a gcs bucket which will be used as storage

`gcloud auth login`

`SERVICE_ACCOUNT_NAME=spinnaker-gcs-account`
`SERVICE_ACCOUNT_DEST=~/.gcp/gcs-account.json`

`gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME --display-name $SERVICE_ACCOUNT_NAME`

`SA_EMAIL=$(gcloud iam service-accounts list --filter="displayName:$SERVICE_ACCOUNT_NAME" --format='value(email)')`

`PROJECT=$(gcloud info --format='value(config.project)')`

`gcloud projects add-iam-policy-binding $PROJECT --role roles/storage.admin --member serviceAccount:$SA_EMAIL`

`mkdir -p $(dirname $SERVICE_ACCOUNT_DEST)`

`gcloud iam service-accounts keys create $SERVICE_ACCOUNT_DEST --iam-account $SA_EMAIL`

`BUCKET_LOCATION=us`

`hal config storage gcs edit --project $PROJECT --bucket-location $BUCKET_LOCATION --json-path $SERVICE_ACCOUNT_DEST`

`hal config storage edit --type gcs`

6.Deploy spinnaker

`hal version list`

`hal config version edit --version 1.15.2`

`sudo hal deploy apply`

Spinnaker UI will be accessible only on ***localhost***. To get the UI we need to do tunnelling.

*UseCase*:
End user does a commit in git, which should trigger the jenkins job , create the docker image and push it to dockerhub. Once this job is
done, it should trigger the spinnaker pipeline which will deploy the image in the kubernetes cluster using helm chart

## Prerequisities:
For the above use case to happen, we need to enable and add accounts for jenkins as ci provider and github as artifact provider

`hal config ci jenkins enable`

`hal config ci jenkins master add my-jenkins --address <jenkins-url> --username <jenkins-username> --password `

We need to genarate a **personal auth token** in github. (Path: Settings-> Developer Settings-> Personal access tokens)
save the genarated token to a file (in this case TOKEN_FILE)

`hal config features edit --artifacts true`

`hal config artifact github enable`

`hal config artifact github account add my-github --token-file TOKEN_FILE`

We should have a repo in github holding *code*, *Dockerfile*, *helm-chart*, ***tar file of helm chart***

To create the helm chart: *helm chart chart-name*

To package: *helm package chart-name*

**Working with Spinnaker UI**
1. We need to create applications to start with
2. Under applications we will be creating our pipelines

**Creating a pipeline**:
1. Once your application is created,u can see *Pipelines*, *Infrastructure*, *Tasks* click on *Pipelines*give a name for it and click on create.
We will setup the pipeline with *Configuration*, *Bake(Manifest)* and *Deploy(Manifest)* stages

2. The first stage will be **Configuration**. 
Select **Expected Artifacts** and provide the paths for *values.yaml* and *helm tar file*. Select Kind as *Github*
To access a github file via api:
Template: https://api.github.com/repos/username/repo_name/contents/file-path

Ex: https://api.github.com/repos/nagarajui7/kube-spin-helm/contents/helloworld-chart/values.yaml
Save the changes.
Then, Select **Automated Triggers** and select *Jenkins* from Type, so that it will listen to Jenkins Jobs
Select master details which we have configured previously from the drop down.
Select the job, which spinnaker should be listening to and click on Save

3. Click on Add Stage and Select **Bake(Manifest)**
Select the Render Engine as *HELM2*. Template artifact as *helm-tar-file* and Overrides as *yaml file*
***NOTE***: Please make sure you select the artifacts from the drop down. If nothing is present please check the expected artifacts.
Provide a name for under **Template Renderer**
In **Produces Artifacts** section and select the kind as *base64* and provide the name same as that of Template Renderer.
Click on Save.

4. Again Click on Add Stage and select **Deploy(Manifest)**.
Select the Kubernetes account we have enabled under **Account** and Manifest artifact as *base64* artifact, which was created in above step.

Now the pipeline is ready. whenever there is a job completed in Jenkins, this pipeline is going to be triggered.
