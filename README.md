# Lightspeed Agent GitOps

ArgoCD deployment configuration for the [Red Hat Lightspeed Agent](https://github.com/RHEcosystemAppEng/google-lightspeed-agent). This repo controls **when and where** the agent is deployed. The Helm charts and application code live in the app repo.

## Structure

```
openshift/
├── application.yaml        # ArgoCD Application CR — syncs the OpenShift Helm chart
└── values-override.yaml    # Environment-specific overrides (image tags, config)
google-cloud/
├── application.yaml        # ArgoCD Application CR — triggers Cloud Build for Cloud Run
└── values-override.yaml    # Environment-specific overrides (project ID, image tags)
```

## How It Works

Each Application CR uses ArgoCD multi-source to combine:
- **This repo** — environment overrides (`values-override.yaml`)
- **App repo** — Helm charts and templates (`deploy/openshift/` or `deploy/gitops/google-cloud/`)

ArgoCD watches both repos. When a change is merged to `main` in either repo, ArgoCD re-renders the Helm chart and syncs.

## Day-2: Updating Image Tags

```bash
# 1. Create a branch
git checkout -b bump-agent-v1.3.0

# 2. Update the image tag
vi openshift/values-override.yaml

# 3. Commit, push, open PR
git add openshift/values-override.yaml
git commit -m "chore: bump agent to v1.3.0"
git push origin bump-agent-v1.3.0
gh pr create --title "Bump agent to v1.3.0"

# 4. SRE reviews and merges → ArgoCD auto-syncs
```

## Initial Setup

1. Install prerequisites on the OpenShift cluster (ArgoCD, ESO for GCP target):
   ```bash
   bash deploy/gitops/setup/install-gitops-operator.sh
   bash deploy/gitops/setup/install-eso-operator.sh      # GCP target only
   bash deploy/gitops/setup/setup-gcp-sa.sh              # GCP target only
   ```
   See [setup docs](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/tree/main/deploy/gitops/setup) in the app repo.

2. Edit the `values-override.yaml` for your target with environment-specific values.

3. Apply the Application CR:
   ```bash
   oc apply -f openshift/application.yaml
   # or
   oc apply -f google-cloud/application.yaml
   ```

## Cross-Cluster Deployment

Deploy the agent from a hub cluster (ArgoCD) to a separate spoke cluster. See also the [cross-cluster reference](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/tree/main/deploy/gitops/README.md#cross-cluster-deployment) in the app repo for credential rotation and ESO details.

### Step 1: Prepare the Spoke Cluster

Log in to the spoke cluster and create a ServiceAccount for ArgoCD:

```bash
oc login <spoke-cluster-api-url>

oc create namespace admin-argocd
oc create sa admin-argocd-sa -n admin-argocd
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:admin-argocd:admin-argocd-sa

# Create a long-lived token
oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argocd-admin-sa-secret
  namespace: admin-argocd
  annotations:
    kubernetes.io/service-account.name: admin-argocd-sa
type: kubernetes.io/service-account-token
EOF

# Save these values — you'll need them on the hub
TOKEN=$(oc get secret argocd-admin-sa-secret -n admin-argocd \
  -o jsonpath='{.data.token}' | base64 -d)
CA=$(oc get secret argocd-admin-sa-secret -n admin-argocd \
  -o jsonpath='{.data.ca\.crt}' | base64)
API_URL=$(oc whoami --show-server)
echo "API_URL: $API_URL"
echo "TOKEN: $TOKEN"
echo "CA: $CA"
```

Create the app secrets on the spoke cluster:

```bash
cp openshift/secrets.yaml.example my-secrets.yaml
vi my-secrets.yaml   # fill in real credentials
oc create namespace lightspeed-agent
oc apply -f my-secrets.yaml -n lightspeed-agent
```

See the [OpenShift deployment docs](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/blob/main/deploy/openshift/README.md#4-configure-values) in the app repo for details on each secret field and configuration options (Vertex AI, persistent sessions, etc.).

> **Note:** This manual approach is for testing and development. For production, use a secrets management solution such as Vault, External Secrets Operator, or SealedSecrets.

### Step 2: Register the Spoke on the Hub Cluster

Log in to the hub cluster and create the ArgoCD cluster connection Secret:

```bash
oc login <hub-cluster-api-url>

# Use the TOKEN, CA, and API_URL from step 1
oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: spoke-cluster
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: spoke-cluster
  server: <API_URL>
  config: |
    {
      "bearerToken": "<TOKEN>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<CA>"
      }
    }
EOF
```

### Step 3: Configure and Apply

Update `openshift/application.yaml` — point `destination.server` to the spoke:

```yaml
  destination:
    server: <API_URL of spoke cluster>
    namespace: lightspeed-agent
```

Update `openshift/values-override.yaml` with your environment values (image tags, providerUrl, etc.).

Commit and push so ArgoCD can read the config:

```bash
git add -A
git commit -m "configure spoke cluster deployment"
git push origin main
```

Apply the Application CR on the hub:

```bash
oc apply -f openshift/application.yaml
```

### Step 4: Verify

```bash
# On the hub — check ArgoCD sync status
oc get application lightspeed-agent -n openshift-gitops \
  -o jsonpath='{.status.sync.status}{" "}{.status.health.status}'
# Expected: Synced Healthy

# On the spoke — check pods are running
oc login <spoke-cluster-api-url>
oc get pods -n lightspeed-agent
```

### Day-2: Update Image Tags

```bash
git checkout -b bump-v1.1.0
vi openshift/values-override.yaml    # change image tags
git add openshift/values-override.yaml
git commit -m "chore: bump agent to v1.1.0"
git push origin bump-v1.1.0
gh pr create --title "Bump agent to v1.1.0"
# SRE reviews and merges → ArgoCD auto-syncs to spoke
```
