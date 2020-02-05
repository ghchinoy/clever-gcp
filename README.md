# clever-gcp

Clever uses of `gcloud`, GCP, and bash to make cli use of GCP easier.


## references

* [Filtering and formatting fun with gcloud, GCP's command line interface](https://cloud.google.com/blog/products/gcp/filtering-and-formatting-fun-with), June 2016
* [gcloud topic filters](https://cloud.google.com/sdk/gcloud/reference/topic/filters), Cloud SDK docs
* [gcloud cheat sheet](https://gist.github.com/pydevops/cffbd3c694d599c6ca18342d3625af97), gist, pydevops 

## project id to bash var

```
export PROJECT_ID=$(gcloud info --format='value(config.project)')
```

Another clever use of the `--format` flag


## obtaining an IP of a Cloud SQL server

```
DB_INSTANCE=flights
MYSQLIP=$(gcloud sql instances describe \
${DB_INSTANCE} --format="value(ipAddresses.ipAddress)")
```

This is clever because it takes the `describe` and parses the response using the built-in `--format` flag

## wait for a service in kubernetes to receive an external IP

```
# clever, wait for service
SERVICENAME=wp-repd-wordpress
while [[ -z $SERVICE_IP ]]; do SERVICE_IP=$(kubectl get svc ${SERVICENAME} -o jsonpath='{.status.loadBalancer.ingress[].ip}'); echo "Waiting for service external IP..."; sleep 2; done; echo http://$SERVICE_IP/admin
```

## wait for a persistent disk in kubernetes to be available
```
# same, for Persistent Disk Volume Claim
SERVICENAME=wp-repd-wordpress
while [[ -z $PV ]]; do PV=$(kubectl get pvc ${SERVICENAME} -o jsonpath='{.spec.volumeName}'); echo "Waiting for PV..."; sleep 2; done
```


## create a service account

Create a service acount with a specific role (roles/storage.admin)

```
export SERVICE_ACCOUNT=user-gcp-sa
export SERVICE_ACCOUNT_EMAIL=${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com
gcloud iam service-accounts create ${SERVICE_ACCOUNT} \
  --display-name "GCP Service Account"
gcloud projects add-iam-policy-binding ${PROJECT_ID} --member \
  serviceAccount:${SERVICE_ACCOUNT_EMAIL} \
  --role=roles/storage.admin
export KEY_FILE=${HOME}/secrets/${SERVICE_ACCOUNT_EMAIL}.json
gcloud iam service-accounts keys create ${KEY_FILE} \
  --iam-account ${SERVICE_ACCOUNT_EMAIL}
export GOOGLE_APPLICATION_CREDENTIALS=${KEY_FILE}
```

## create 3 vms

Create 3 vms, with a particular image (bitnam's nginx)

```
for i in {1..3}; \
do \
  gcloud compute instances create "nginxstack-$i" \
  --machine-type "f1-micro" \
  --tags nginxstack-tcp-443,nginxstack-tcp-80 \
  --zone us-central1-f \
  --image   "https://www.googleapis.com/compute/v1/projects/bitnami-launchpad/global/images/bitnami-nginx-1-14-0-4-linux-debian-9-x86-64" \
  --boot-disk-size "200" --boot-disk-type "pd-standard" \
  --boot-disk-device-name "nginxstack-$i"; \
Done
```

## get latest valid kubernetes version for a region

```
gcloud container get-server-config --region us-west1 --format='value(validMasterVersions[0])'
```

assign this version to an env var

```
CLUSTER_VERSION=$(gcloud container get-server-config --region us-west1 --format='value(validMasterVersions[0])')
```

## create a non-API V1 nultinode, regional k8s cluster

using `CLUSTER_VERSION` from above

```
export CLOUDSDK_CONTAINER_USE_V1_API_CLIENT=false
export CLUSTER_NAME=repd
# multinode cluster
gcloud container clusters create ${CLUSTER_NAME} \
  --cluster-version=${CLUSTER_VERSION} \
  --machine-type=n1-standard-4 \
  --region=us-west1 \
  --num-nodes=1 \
  --node-locations=us-west1-a,us-west1-b,us-west1-c
```

## enable services 

use `gcloud services enable SERVICENAME` after getting a list of services from `gcloud services list`, such as

```
gcloud services enable deploymentmanager.googleapis.com servicemanagement.googleapis.com container.googleapis.com cloudresourcemanager.googleapis.com endpoints.googleapis.com file.googleapis.com ml.googleapis.com iam.googleapis.com sqladmin.googleapis.com 
```

## create a firewall rule

using network tag `customaccess`, create a firewall rule called `allow-custom`:

```
gcloud compute \
--project=$PROJECT_ID \
firewall-rules create allow-custom \
--direction=INGRESS \
--priority=1000 \
--network=default \
--action=ALLOW \
--rules=tcp:9870,tcp:8088,tcp:8080 \
--source-ranges=$BROWSER_IP/32 \
--target-tags=customaccess
```

# Custom Dataproc cluster

install custom software onto a Dataproc cluster using a precreated, gcs-stored, bash script as well as a publically-available demo sample:

```
gcloud dataproc clusters create cluster-custom \
--bucket $BUCKET \
--subnet default \
--zone $MYZONE \
--region $MYREGION \
--master-machine-type n1-standard-2 \
--master-boot-disk-size 100 \
--num-workers 2 \
--worker-machine-type n1-standard-1 \
--worker-boot-disk-size 50 \
--num-preemptible-workers 2 \
--image-version 1.2 \
--scopes 'https://www.googleapis.com/auth/cloud-platform' \
--tags customaccess \
--project $PROJECT_ID \
--initialization-actions 'gs://'$BUCKET'/init-script.sh','gs://cloud-training-demos/dataproc/datalab.sh'
```

example `init-script.sh`

```
#!/bin/bash

# install Google Python client on all nodes
apt-get update
apt-get install -y python-pip
pip install --upgrade google-api-python-client

ROLE=$(/usr/share/google/get_metadata_value attributes/dataproc-role)
if [[ "${ROLE}" == 'Master' ]]; then
   git clone https://github.com/GoogleCloudPlatform/training-data-analyst
fi
```

# vm recommender for all projects

see `recommender` bash script by Rishi Singhal
