# 🛡️ GKE + External Secrets Operator + Tailscale Operator with Workload Identity Federation (WIF)

This guide walks you through setting up a Google Kubernetes Engine (GKE) cluster that uses **Workload Identity Federation (WIF)** to integrate the [External Secrets Operator (ESO)](https://external-secrets.io) and the [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator).

No static service account keys are used. Instead, we use Kubernetes Service Accounts (KSAs) with WIF to access Google Secret Manager securely.

---

## 📦 Required Files

- `external-secret-store.yml` — creates a `SecretStore` that uses GCP Secret Manager
- `secretstore.yml` — defines an `ExternalSecret` that mounts a GCP secret as a Kubernetes secret

---

## 1. Environment Setup

```bash
export PROJECT_ID="noble-anvil-460215-s8"
export PROJECT_NUM="262274137731"
export CLUSTER_NAME="gke-eso"
export CLUSTER_LOCATION="us-central1-c"
export WORKLOAD_POOL="${PROJECT_ID}.svc.id.goog"

export KSA_NAMESPACE="tailscale"
export KSA_NAME="external-secrets-v2"

export TAILSCALE_CLIENT_ID="your_client_id_here"
export TAILSCALE_CLIENT_SECRET="your_client_secret_here"
```

Ensure APIs are enabled:

```bash
gcloud services enable container.googleapis.com iamcredentials.googleapis.com secretmanager.googleapis.com
```

---

## 2. Create GKE Cluster with Workload Identity Enabled

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $CLUSTER_LOCATION \
  --workload-pool=$WORKLOAD_POOL \
  --num-nodes 2 --machine-type e2-medium \
  --enable-ip-alias --project $PROJECT_ID

gcloud container clusters get-credentials $CLUSTER_NAME \
  --zone $CLUSTER_LOCATION --project $PROJECT_ID
```

---

## 3. Reserve Namespace and Install ESO

```bash
kubectl create namespace $KSA_NAMESPACE

helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace $KSA_NAMESPACE \
  --set installCRDs=true \
  --set serviceAccount.create=true \
  --set serviceAccount.name=$KSA_NAME
```

Verify:

```bash
kubectl get deployment -n $KSA_NAMESPACE
kubectl get crds | grep externalsecrets
```

---

## 4. Create Secrets in Secret Manager

```bash
echo -n "$TAILSCALE_CLIENT_ID" | gcloud secrets create tailscale-oauth-client-id \
  --replication-policy="automatic" --data-file=- --project $PROJECT_ID

echo -n "$TAILSCALE_CLIENT_SECRET" | gcloud secrets create tailscale-oauth-client-secret \
  --replication-policy="automatic" --data-file=- --project $PROJECT_ID
```

---

## 5. Grant secretAccessor Permissions to KSA

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="principal://iam.googleapis.com/projects/${PROJECT_NUM}/locations/global/workloadIdentityPools/${WORKLOAD_POOL}/subject/ns/${KSA_NAMESPACE}/sa/${KSA_NAME}" \
  --role="roles/secretmanager.secretAccessor"
```

---

## 6. Deploy SecretStore & ExternalSecret

### `external-secret-store.yml`

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: tailscale-store
  namespace: ${KSA_NAMESPACE}
spec:
  provider:
    gcpsm:
      projectID: ${PROJECT_ID}
```

Apply:

```bash
envsubst < external-secret-store.yml | kubectl apply -f -
```

### `secretstore.yml`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: operator-oauth
  namespace: ${KSA_NAMESPACE}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: tailscale-store
    kind: SecretStore
  target:
    name: operator-oauth
    creationPolicy: Owner
  data:
    - secretKey: client_id
      remoteRef:
        key: tailscale-oauth-client-id
    - secretKey: client_secret
      remoteRef:
        key: tailscale-oauth-client-secret
```

Apply:

```bash
envsubst < secretstore.yml | kubectl apply -f -
```

Wait for it to sync:

```bash
kubectl wait externalsecret operator-oauth -n $KSA_NAMESPACE --for=condition=Ready=True --timeout=60s
```

Check:

```bash
kubectl get secret operator-oauth -n $KSA_NAMESPACE -o yaml
```

---

## 7. Install Tailscale Operator (with ExternalSecret-managed credentials)

```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update

helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  --namespace $KSA_NAMESPACE \
  --set oauth.clientSecret.existingSecret=operator-oauth \
  --set oauth.clientSecret.key=client_secret \
  --set oauth.clientId.existingSecret=operator-oauth \
  --set oauth.clientId.key=client_id \
  --create-namespace
```

Rollout restart if secret was added after operator:

```bash
kubectl rollout restart deployment -n $KSA_NAMESPACE tailscale-operator
```

---

## ✅ Validation

You should see a pod named `tailscale-operator`:

```bash
kubectl get pods -n $KSA_NAMESPACE
```

Then go to [Tailscale Admin Console](https://login.tailscale.com/admin/machines) and look for:

- `tailscale-operator` node
- `tag:k8s-operator` tag

---

## Summary

| Step                        | Purpose                              |
| --------------------------- | ------------------------------------ |
| Enable Workload Identity    | Avoids long-lived GSA keys           |
| ESO + WIF                   | Securely fetches secrets without GSA |
| Tailscale OAuth Secrets     | Provided via Secret Manager          |
| ExternalSecret              | Syncs Tailscale secrets into K8s     |
| Helm-based Operator Install | Uses synced secrets at runtime       |
# gke-tailscale-operator
