# TPU Provisioner Deployment Guide

This document describes how to deploy the `tpu-provisioner` to a GKE cluster.

## Prerequisites: Workload Identity Setup

The TPU Provisioner requires Workload Identity to interact with GCP APIs. Follow these steps to configure the necessary Google Service Account (GSA) and bind it to the Kubernetes Service Account (KSA).

### 1. Define Variables

Ensure your `gcloud` context is set to the correct project:

```bash
PROJECT_ID=$(gcloud config get-value project)
if [ -z "${PROJECT_ID}" ]; then
  echo "PROJECT_ID is not set. Please run: gcloud config set project <PROJECT_ID>"
  exit 1
fi

NAMESPACE="tpu-provisioner-system"
KSA_NAME="tpu-provisioner-controller-manager"
GSA_NAME="tpu-provisioner"
GSA_EMAIL="${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
```

### 2. Enable Google Cloud APIs

```bash
gcloud services enable container.googleapis.com \
    tpu.googleapis.com \
    logging.googleapis.com \
    monitoring.googleapis.com
```

### 3. Create the Google Service Account

```bash
gcloud iam service-accounts create "${GSA_NAME}" \
    --display-name="TPU Provisioner Service Account"
```

### 4. Grant IAM Roles

The TPU Provisioner requires the following roles:

- `roles/monitoring.metricWriter`
- `roles/logging.logWriter`
- `roles/container.clusterAdmin`
- `roles/iam.serviceAccountUser`

```bash
ROLES=(
    "roles/monitoring.metricWriter"
    "roles/logging.logWriter"
    "roles/container.clusterAdmin"
    "roles/iam.serviceAccountUser"
)

for ROLE in "${ROLES[@]}"; do
    gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
        --member="serviceAccount:${GSA_EMAIL}" \
        --role="${ROLE}"
done
```

### 5. Bind KSA to GSA

```bash
gcloud iam service-accounts add-iam-policy-binding "${GSA_EMAIL}" \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${NAMESPACE}/${KSA_NAME}]"
```

### 6. Annotate the Kubernetes Service Account

```bash
kubectl annotate serviceaccount -n "${NAMESPACE}" "${KSA_NAME}" \
    iam.gke.io/gcp-service-account="${GSA_EMAIL}"
```

---

## Deployment with Kustomize

The deployment is managed using Kustomize. You can use the `config/default` or `config/default-slice` directories as a base for your deployment.

### 1. Update the Controller Image

Before deploying, update the image to point to your specific container registry and tag.

```bash
cd config/default
kustomize edit set image controller=us-docker.pkg.dev/YOUR_PROJECT/YOUR_REPOSITORY/tpu-provisioner:YOUR_TAG
```

### 2. Customize Configuration (Optional)

You can customize the controller's behavior by modifying the ConfigMap in `config/manager/configmap.yaml` or by providing a patch. For example, if using the `default-slice` overlay, you can modify `config/default-slice/config_patch.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tpu-provisioner-manager
  namespace: tpu-provisioner-system
data:
  ENABLE_SLICE_CONTROLLER: "true"
  ENABLE_WEBHOOKS: "true"
  GCP_ZONE: "us-central1-a"
```

### 3. Apply the Configuration

Apply the configuration to your cluster using server-side apply:

```bash
kubectl apply --server-side -k config/default
```

---

## Creating a Cluster-Specific Overlay

For multi-cluster environments or staging/production separation, it is recommended to create a cluster-specific Kustomize overlay. This allows you to manage unique configurations (e.g., image tags, GCP zone, feature flags) for each environment.

### 1. Create a New Directory

Create a directory under `config/` named after your cluster or environment:

```bash
CLUSTER_NAME="my-tpu-cluster"
mkdir -p config/${CLUSTER_NAME}
```

### 2. Create `kustomization.yaml`

Define your overlay by referencing the base configuration and adding cluster-specific overrides.

```yaml
# config/${CLUSTER_NAME}/kustomization.yaml
resources:
- ../default

patchesStrategicMerge:
- config_patch.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
- name: controller
  newName: us-docker.pkg.dev/YOUR_PROJECT/YOUR_REPOSITORY/tpu-provisioner
  newTag: YOUR_TAG
```

### 3. Create `config_patch.yaml`

Use this file to override settings in the `tpu-provisioner-manager` ConfigMap for this specific cluster.

```yaml
# config/${CLUSTER_NAME}/config_patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tpu-provisioner-manager
  namespace: tpu-provisioner-system
data:
  ENABLE_SLICE_CONTROLLER: "true"
  ENABLE_WEBHOOKS: "true"
  GCP_ZONE: "us-central1-a"
```

### 4. Deploy to Your Cluster

Apply the cluster-specific configuration:

```bash
kubectl apply --server-side -k config/${CLUSTER_NAME}
```

---

## Verify the Deployment

Ensure the controller manager is running:

```bash
kubectl get pods -n tpu-provisioner-system
```

If you made IAM changes or configMap changes recently, you might need to restart the pods:

```bash
kubectl rollout restart deployment -n tpu-provisioner-system tpu-provisioner-controller-manager
```
