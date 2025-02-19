# Apigee hybrid Runtime setup

Where? 

In Google Cloud, and we create a GCP project for this, plus a GKE cluster...

<your_gcp_project_id>

## Versions

Version:  1.14.0
- Doc:    [text](https://cloud.google.com/apigee/docs/hybrid/v1.14/what-is-hybrid)

Version:	1.13.1
- Doc:		[text](https://cloud.google.com/apigee/docs/hybrid/v1.13/what-is-hybrid)

Release Notes: [text](https://cloud.google.com/apigee/docs/hybrid/release-notes)

Important:

- GCP project: <your_gcp_project_id>
- All commands are executed from Cloud Shell

## START Apigee hybrid setup (latest version)

``` 
gcloud auth login 
```

```
mkdir apigee-hybrid-1.14.0
cd apigee-hybrid-1.14.0
```

``` 
export PROJECT_ID=<your_gcp_project_id>
```

Create a GKE cluster that respects the requirements for non-prod:
https://cloud.google.com/apigee/docs/hybrid/v1.13/cluster-overview#minimum-configurations

# gcloud commands:

# Create the cluster (n2-standard-4)

The cluster we create is regional (europe-west1)

* Number of nodes: ```6```
* Total vCPUs: ```24```
* Total Memory: ```96 GB```
* Workload identity: ```yes```
* Node pools: ```2```
  - ```apigee-data```
  - ```apigee-runtime```

```
gcloud container clusters create "apigee-hybrid-wingman" \
  --region "europe-west1" \
  --machine-type "n2-standard-4" \
  --num-nodes "2" \
  --cluster-version "1.30" \
  --enable-ip-alias \
  --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
  --disk-size "100" \
  --disk-type "pd-standard" \
  --no-enable-basic-auth \
  --no-issue-client-certificate \
  --metadata disable-legacy-endpoints=true \
  --workload-pool="<your_gcp_project_id>.svc.id.goog" 
```

## Workload Identity

Doc:	https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity

# Create the "apigee-data" node pool
```
gcloud container node-pools create "apigee-data" \
  --cluster "apigee-hybrid-wingman" \
  --region "europe-west1" \
  --machine-type "n2-standard-4" \
  --num-nodes "1"
```

# Create the "apigee-runtime" node pool
```
gcloud container node-pools create "apigee-runtime" \
  --cluster "apigee-hybrid-wingman" \
  --region "europe-west1" \
  --machine-type "n2-standard-4" \
  --num-nodes "1"
```
  
# Delete the "default-pool" node pool using the following gcloud command:
```
gcloud container node-pools delete "default-pool" \
  --cluster "apigee-hybrid-wingman" \
  --region "europe-west1"
```

#################

## Part 1: Focus on Control Plane

### Enable APIS
```
gcloud services enable \
    apigee.googleapis.com \
    apigeeconnect.googleapis.com \
    cloudapis.googleapis.com \
    cloudresourcemanager.googleapis.com \
    compute.googleapis.com \
    dns.googleapis.com \
    iam.googleapis.com \
    iamcredentials.googleapis.com \
    pubsub.googleapis.com \
    servicemanagement.googleapis.com \
    serviceusage.googleapis.com \
    storage-api.googleapis.com \
    storage-component.googleapis.com  --project $PROJECT_ID
```

```
gcloud services list --project $PROJECT_ID
```

### Create apigee org

```
export TOKEN=$(gcloud auth print-access-token)
export ORG_NAME=$PROJECT_ID
export ANALYTICS_REGION="europe-west1"
export RUNTIMETYPE=HYBRID
```

### No data residency:

```
curl -H "Authorization: Bearer $TOKEN" -X POST -H "content-type:application/json" \
  -d '{
    "name":"'"$ORG_NAME"'",
    "runtimeType":"'"$RUNTIMETYPE"'",
    "analyticsRegion":"'"$ANALYTICS_REGION"'"
  }' \
  "https://apigee.googleapis.com/v1/organizations?parent=projects/$PROJECT_ID"
```

### Retrieve information about the apigee org:
```
curl -H "Authorization: Bearer $TOKEN" \
  "https://apigee.googleapis.com/v1/organizations/$ORG_NAME"
```

### Create env + env group + attachment (env attached to env group)

```
export ENV_NAME="dev"

curl -H "Authorization: Bearer $TOKEN" -X POST -H "content-type:application/json"   -d '{
    "name": "'"$ENV_NAME"'"
  }'   "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/environments"

curl -H "Authorization: Bearer $TOKEN" \
          "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/environments"
```

```
export DOMAIN="<your_domain>" # e.g. hybrid.iloveapis.io
```

```
export ENV_GROUP="dev-group"

curl -H "Authorization: Bearer $TOKEN" -X POST -H "content-type:application/json" \
   -d '{
     "name": "'"$ENV_GROUP"'",
     "hostnames":["'"$DOMAIN"'"]
   }' \
   "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/envgroups"
    

curl -H "Authorization: Bearer $TOKEN" \
  "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/envgroups"


curl -H "Authorization: Bearer $TOKEN" -X POST -H "content-type:application/json" \
   -d '{
     "environment": "'"$ENV_NAME"'",
   }' \
   "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/envgroups/$ENV_GROUP/attachments"
 

curl -H "Authorization: Bearer $TOKEN" \
  "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/envgroups/$ENV_GROUP/attachments"
```

#################

## Part 2: Focus on Runtime Plane

```
export CLUSTER_NAME="apigee-hybrid-wingman"
export CLUSTER_LOCATION="europe-west1"
export PROJECT_ID="<your_gcp_project_id>"

gcloud container node-pools list \
--cluster=${CLUSTER_NAME} \
--region=${CLUSTER_LOCATION} \
--project=${PROJECT_ID}
```

### Get cluster credentials
```
gcloud container clusters get-credentials ${CLUSTER_NAME} \
--region ${CLUSTER_LOCATION} \
--project ${PROJECT_ID}
```

## Get the name of the current default StorageClass
```
kubectl get sc
```
### Create new file storageclass.yaml
```
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: "apigee-sc"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: none
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```
```
kubectl apply -f storageclass.yaml
```
### Change the storage class
```
kubectl patch storageclass standard-rwo \
-p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl patch storageclass apigee-sc \
-p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Verify apigee-sc is the default storage
```
kubectl get sc
```
### Verify workload identity has been enabled 
```
gcloud container clusters describe ${CLUSTER_NAME} \
  --region ${CLUSTER_LOCATION} \
  --project ${PROJECT_ID} | grep -i "workload"
```

#### HELM CHARTS ####
```
mkdir apigee-hybrid

mkdir apigee-hybrid/helm-charts

cd apigee-hybrid/helm-charts

export APIGEE_HELM_CHARTS_HOME=$PWD

export CHART_REPO=oci://us-docker.pkg.dev/apigee-release/apigee-hybrid-helm-charts
export CHART_VERSION=1.13.1
helm pull $CHART_REPO/apigee-operator --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-datastore --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-env --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-ingress-manager --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-org --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-redis --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-telemetry --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-virtualhost --version $CHART_VERSION --untar
```

### Install tree
```
sudo apt-get install tree
```
### Verify that the charts expanded into the intended directory structure
```
tree
```
### Create the apigee namespace
```
kubectl create namespace apigee

kubectl get namespace apigee
```

### Create Service Accounts (prod)

```
$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account --help

chmod +x $APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account
```

### Command to create the service accounts:
```
$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-cassandra \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-datastore

$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-logger \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-telemetry

$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-mart \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-org

$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-metrics \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-telemetry

$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-runtime \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-env

$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-synchronizer \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-env

$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-udca \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-org

$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --profile apigee-watcher \
  --env prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-org
```

### Copy the apigee-udca JSON file to the apigee-env chart directory
```
cp $APIGEE_HELM_CHARTS_HOME/apigee-org/$PROJECT_ID-apigee-udca.json $APIGEE_HELM_CHARTS_HOME/apigee-env/
```
## Verification
```
tree -P *.json
```
### Create TLS Certificate
```
mkdir $APIGEE_HELM_CHARTS_HOME/apigee-virtualhost/certs/

export DOMAIN=*.nip.io

openssl req  -nodes -new -x509 -keyout $APIGEE_HELM_CHARTS_HOME/apigee-virtualhost/certs/keystore_$ENV_GROUP.key -out \
    $APIGEE_HELM_CHARTS_HOME/apigee-virtualhost/certs/keystore_$ENV_GROUP.pem -subj '/CN='$DOMAIN'' -days 3650 \
    -addext "subjectAltName = DNS:*.nip.io"

ls $APIGEE_HELM_CHARTS_HOME/apigee-virtualhost/certs
```

## Overrides.yaml file

### To get the list of service account emails
```
gcloud iam service-accounts list --project ${PROJECT_ID} --filter "apigee"
```

cf. [overrides.template.yaml](overrides.template.yaml)

```
cd $APIGEE_HELM_CHARTS_HOME

git clone https://github.com/JoelGauci/apigee-hybrid-setup.git

mv ./apigee-hybrid-setup/overrides.template.yaml ../

echo $PROJECT_ID

envsubst < overrides.template.yaml > overrides.yaml
```

### Verify PROJECT_ID env variable has been substituted

```
vi overrides.yaml

rm overrides.template.yaml
```

## Enable synchronyzer access

### Verify you have the Apigee Admin role
```
gcloud projects get-iam-policy ${PROJECT_ID}  \
  --flatten="bindings[].members" \
  --format='table(bindings.role)' \
  --filter="bindings.members:<your_email_address>"
```
### ... If not ....
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member user:<your_email_address> \
  --role roles/apigee.admin
```

### Enable Synchronizer access using the Apigee API

```
export TOKEN=$(gcloud auth print-access-token)

gcloud iam service-accounts list --project ${PROJECT_ID} --filter "apigee-synchronizer"

curl -X POST -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type:application/json" \
  "https://apigee.googleapis.com/v1/organizations/<your_gcp_project_id>:setSyncAuthorization" \
   -d '{"identities":["'"serviceAccount:apigee-synchronizer@<your_gcp_project_id>.iam.gserviceaccount.com"'"]}'

curl -X GET -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type:application/json" \
  "https://apigee.googleapis.com/v1/organizations/<your_gcp_project_id>:getSyncAuthorization"
```   

## Enable analytics publisher access

```
curl -X  PATCH -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type:application/json" \
  "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/controlPlaneAccess?update_mask=analytics_publisher_identities" \
  -d "{\"analytics_publisher_identities\": [\"serviceAccount:apigee-runtime@$ORG_NAME.iam.gserviceaccount.com\"]}"


curl -X GET -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
  -H "Content-Type:application/json"  \
  "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/operations/<LONGRUNNING_ID>"
```

## Verify the organization's ControlPlaneAccess configuration

```
curl "https://apigee.googleapis.com/v1/organizations/$ORG_NAME/controlPlaneAccess" \
-H "Authorization: Bearer $(gcloud auth print-access-token)"
```

...you must see a response like this one:

```
{
  "synchronizerIdentities": [
    "serviceAccount:apigee-synchronizer@<your-project-id>.iam.gserviceaccount.com"
  ],
  "analyticsPublisherIdentities": [
    "serviceAccount:apigee-runtime@<your-project-id>.iam.gserviceaccount.com"
  ]
}
```

## START INSTALLATION (Cert Manager + CRDs)

### Cert manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.1/cert-manager.yaml

kubectl get all -n cert-manager -o wide
```
### Install CRDs
```
kubectl apply -k  apigee-operator/etc/crds/default/ \
  --server-side \
  --force-conflicts \
  --validate=false \
  --dry-run=server

kubectl apply -k  apigee-operator/etc/crds/default/ \
  --server-side \
  --force-conflicts \
  --validate=false

kubectl get crds | grep apigee
```

...you must see a the following CRDs:

```
apigeedatastores.apigee.cloud.google.com               2025-02-19T11:05:34Z
apigeedeployments.apigee.cloud.google.com              2025-02-19T11:05:36Z
apigeeenvironments.apigee.cloud.google.com             2025-02-19T11:05:38Z
apigeeissues.apigee.cloud.google.com                   2025-02-19T11:05:39Z
apigeeorganizations.apigee.cloud.google.com            2025-02-19T11:05:41Z
apigeeredis.apigee.cloud.google.com                    2025-02-19T11:05:44Z
apigeerouteconfigs.apigee.cloud.google.com             2025-02-19T11:05:44Z
apigeeroutes.apigee.cloud.google.com                   2025-02-19T11:05:45Z
apigeetelemetries.apigee.cloud.google.com              2025-02-19T11:05:48Z
cassandradatareplications.apigee.cloud.google.com      2025-02-19T11:05:53Z
secretrotations.apigee.cloud.google.com                2025-02-19T11:05:53Z
```

## START INSTALLATION (HELM Charts)
```
cd $APIGEE_HELM_CHARTS_HOME
```
## Install helm 3.10+
```
sudo apt-get remove helm
rm -rf ~/.helm 
sudo rm /usr/local/bin/helm

curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

sudo cp /usr/sbin/helm /usr/local/bin/helm
```

### Using helm for apigee hybrid install

#### operator / controller

```
helm upgrade operator apigee-operator/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml \
  --dry-run

helm upgrade operator apigee-operator/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml

helm ls -n apigee

kubectl -n apigee get deploy apigee-controller-manager
```
#### datastore
```
helm upgrade datastore apigee-datastore/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml \
  --dry-run=server

helm upgrade datastore apigee-datastore/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml

kubectl -n apigee get apigeedatastore default

kubectl get pods -n apigee # to get the list of pods and verify the status, access logs...
```
#### For each step : if state = creating then go on waiting for the finalization...

#### telemetry
```
helm upgrade telemetry apigee-telemetry/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml \
  --dry-run

helm upgrade telemetry apigee-telemetry/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml

kubectl -n apigee get apigeetelemetry apigee-telemetry
```

#### redis
```
helm upgrade redis apigee-redis/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml \
  --dry-run

helm upgrade redis apigee-redis/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml
```

#### ingress manager
```
helm upgrade ingress-manager apigee-ingress-manager/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml \
  --dry-run

helm upgrade ingress-manager apigee-ingress-manager/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml

kubectl -n apigee get deployment apigee-ingressgateway-manager
```

#### organization
```
helm upgrade $ORG_NAME apigee-org/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml \
  --dry-run

helm upgrade $ORG_NAME apigee-org/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml
```

>> Watch Out: OUTPUT TO EXECUTE for workload identity

```
kubectl -n apigee get apigeeorg
```

#### env
```
helm upgrade $ENV_NAME apigee-env/ \
  --install \
  --namespace apigee \
  --atomic \
  --set env=$ENV_NAME \
  -f overrides.yaml \
  --dry-run

helm upgrade $ENV_NAME apigee-env/ \
  --install \
  --namespace apigee \
  --atomic \
  --set env=$ENV_NAME \
  -f overrides.yaml
```

>> Watch Out: OUTPUT TO EXECUTE for workload identity
```
kubectl -n apigee get apigeeenv
```

#### envgroup / virtualhost
```
helm upgrade $ENV_GROUP apigee-virtualhost/ \
  --install \
  --namespace apigee \
  --atomic \
  --set envgroup=$ENV_GROUP \
  -f overrides.yaml \
  --dry-run

helm upgrade $ENV_GROUP apigee-virtualhost/ \
  --install \
  --namespace apigee \
  --atomic \
  --set envgroup=$ENV_GROUP \
  -f overrides.yaml

kubectl -n apigee get arc

kubectl -n apigee get ar
```

### Workload Identity on GKE
```
gcloud container clusters describe $CLUSTER_NAME \
  --region $CLUSTER_LOCATION \
  --project $PROJECT_ID \
  --flatten 'workloadIdentityConfig'
```
>> check:
```
---
workloadPool: <your-project-id>.svc.id.goog
```

```
gcloud container node-pools describe apigee-runtime \
  --cluster $CLUSTER_NAME \
  --region $CLUSTER_LOCATION \
  --project $PROJECT_ID \
  --flatten "config:"
```

>> check:
```
workloadMetadataConfig:
  mode: GKE_METADATA
```

```
gcloud container node-pools describe apigee-data \
  --cluster $CLUSTER_NAME \
  --region $CLUSTER_LOCATION \
  --project $PROJECT_ID \
  --flatten "config:"
```
>> check:

```
workloadMetadataConfig:
  mode: GKE_METADATA
```

### Apigee ingress

Using GKE 

```kubectl get svc -n apigee -l app=apigee-ingressgateway```

...note the IP address: <lb_ip>

## Next Step

Create an API proxy and test !

Mock:

base path: ```/mock```

target: ```https://mocktarget.apigee.net```

curl command:
```
curl https://${DOMAIN}/mock -k -v --resolve ${DOMAIN}:443:<lb_ip>
```

Better for complete tests:

- create a developer
- create an API product
- create a developer app
- add an API Key Verification policy the original proxy that was created

Developer app:
```
export API_KEY=<consumerKey>
```

API Key Verification policy:

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<VerifyAPIKey name="VA-VerifyAPIKey">
  <APIKey ref="request.header.x-apikey"/>
</VerifyAPIKey>
```

curl command:
```
curl https://${DOMAIN}/mock \
-H "x-apikey: ${API_KEY}"  \
-k \
-v \
--resolve ${DOMAIN}:443:<lb_ip>
```
