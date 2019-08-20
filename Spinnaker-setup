#####Steps for installing Spinnaker#####
1. Install Halyard
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
run it as ubuntu user
sudo bash InstallHalyard.sh
check the version using: hal -v

2.Setting up Docker-Registry Cloud Provider
hal config provider docker-registry enable
hal config provider docker-registry account add my-docker --address index.docker.io --repositories 	library/nginx --username nagarajubatchu1 --password

3. Setting up Kubernetes Cloud Provider
Note: Before enabling kubernetes provider, make sure .kube directory is present along with config file in it.
hal config provider kubernetes enable
hal config provider kubernetes account add my-k8s --repositories my-docker
CONTEXT=kubectl config current-context
hal config provider kubernetes account add my-k8s-3 --provider-version v2 --context $CONTEXT

4. Environment Type
hal config deploy edit --type localdebian

5. Setup Storage GCS
We will be creating an iam role and a auth token for spinning a gcs bucket which will be used as storage
gcloud auth login
SERVICE_ACCOUNT_NAME=spinnaker-gcs-account
SERVICE_ACCOUNT_DEST=~/.gcp/gcs-account.json
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME --display-name $SERVICE_ACCOUNT_NAME
SA_EMAIL=$(gcloud iam service-accounts list --filter="displayName:$SERVICE_ACCOUNT_NAME" --format='value(email)')
PROJECT=$(gcloud info --format='value(config.project)')
gcloud projects add-iam-policy-binding $PROJECT --role roles/storage.admin --member serviceAccount:$SA_EMAIL
mkdir -p $(dirname $SERVICE_ACCOUNT_DEST)
gcloud iam service-accounts keys create $SERVICE_ACCOUNT_DEST --iam-account $SA_EMAIL
BUCKET_LOCATION=us
hal config storage gcs edit --project $PROJECT --bucket-location $BUCKET_LOCATION --json-path $SERVICE_ACCOUNT_DEST
hal config storage edit --type gcs

6.Deploy spinnaker
hal version list
hal config version edit --version 1.15.2
sudo hal deploy apply

Spinnaker UI will be accessible only on localhost. To get the UI we need to do tunnelling.

UseCase:
End user does a commit in git, which should trigger the jenkins job , create the docker image and push it to dockerhub. Once this job is
done, it should trigger the spinnaker pipeline which will deploy the image in the kubernetes cluster

**Prerequisities**:
For the above use case to happen, we need to enable and add accounts for jenkins as ci provider and github as artifact provider
hal config ci jenkins enable
hal config ci jenkins master add my-jenkins --address <jenkins-url> --username <jenkins-username> --password 

We need to genarate a personal auth token in github. (Path: Settings-> Developer Settings-> Personal access tokens)
save the genarated token to a file (in this case TOKEN_FILE)
hal config features edit --artifacts true
hal config artifact github enable
hal config artifact github account add my-github --token-file TOKEN_FILE

