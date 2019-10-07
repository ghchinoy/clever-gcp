# clever-gcp

Clever uses of `gcloud`, GCP, and bash to make cli use of GCP easier.


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
