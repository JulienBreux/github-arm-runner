# 🏃🏼‍♂️‍➡️ Github ARM runner

```sh
export GITHUB_TOKEN=

export PROJECT_ID=

export NETWORK_NAME=runner-network

export IMAGE_NAME=github-runner-arm
export IMAGE_STABLE_TAG=stable

export GCP_SERVICE_ACCOUNT_NAME=github-runner
export GCP_SERVICE_ACCOUNT_ROLE=roles/editor

export GKE_CLUSTER_NAME=github-runner-cluster
export GKE_CLUSTER_REGION=us-central1
export GKE_NAMESPACE=default
export GKE_SERVICE_ACCOUNT_NAME=github-runner
export GKE_SECRET_NAME=github-token

# Select current project
gcloud config set project ${PROJECT_ID}

# Enable APIs
gcloud services enable container.googleapis.com \
containerregistry.googleapis.com \
cloudbuild.googleapis.com

# Network creation
gcloud compute networks create ${NETWORK_NAME} \
--subnet-mode=auto

# Build default image
gcloud builds submit --tag gcr.io/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_STABLE_TAG} .

# Create GKE autopilot cluster
gcloud beta container clusters create ${GKE_CLUSTER_NAME} \
--release-channel regular \
--workload-pool=${PROJECT_ID}.svc.id.goog \
--region ${GKE_CLUSTER_REGION}

# Connect to GKE cluster
gcloud container clusters get-credentials ${GKE_CLUSTER_NAME}

# Create GCP service account and retreive email
gcloud iam service-accounts create ${GCP_SERVICE_ACCOUNT_NAME} --display-name=${GCP_SERVICE_ACCOUNT_NAME}
export GCP_SERVICE_ACCOUNT_EMAIL=$(gcloud iam service-accounts list --filter="displayName:${GCP_SERVICE_ACCOUNT_NAME}" --format='value(email)')

# Grant GCP service account
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member serviceAccount:${GCP_SERVICE_ACCOUNT_EMAIL} \
--role ${GCP_SERVICE_ACCOUNT_ROLE}

# Create GKE service account
kubectl create serviceaccount ${GKE_SERVICE_ACCOUNT_NAME} \
-n ${GKE_NAMESPACE}

# Bind GKE service account to GCP service account
gcloud iam service-accounts add-iam-policy-binding \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[${GKE_NAMESPACE}/${GKE_SERVICE_ACCOUNT_NAME}]" \
${GCP_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com

# Bind GCP service account to GKE service account
kubectl annotate serviceaccount \
${GKE_SERVICE_ACCOUNT_NAME} \
iam.gke.io/gcp-service-account=${GCP_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
-n ${GKE_NAMESPACE}

# Create GKE secret /!\ Replace by new secret manager
kubectl create secret generic ${GKE_SECRET_NAME} \
--from-literal=GITHUB_TOKEN=${GITHUB_TOKEN} \
-n ${GKE_NAMESPACE}
```
