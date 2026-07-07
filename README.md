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
oc login $OCP_SPOKE_SERVER

oc create namespace rh-lightspeed-agent-argocd
oc create sa rh-lightspeed-agent-argocd-sa -n rh-lightspeed-agent-argocd
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:rh-lightspeed-agent-argocd:rh-lightspeed-agent-argocd-sa

# Create a long-lived token
oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argocd-admin-sa-secret
  namespace: rh-lightspeed-agent-argocd
  annotations:
    kubernetes.io/service-account.name: rh-lightspeed-agent-argocd-sa
type: kubernetes.io/service-account-token
EOF

# Save these values — you'll need them on the hub
TOKEN=$(oc get secret argocd-admin-sa-secret -n rh-lightspeed-agent-argocd \
  -o jsonpath='{.data.token}' | base64 -d)
CA=$(oc get secret argocd-admin-sa-secret -n rh-lightspeed-agent-argocd \
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
oc login $OCP_HUB_SERVER

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
    server: $OCP_SPOKE_SERVER
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
oc login $OCP_SPOKE_SERVER
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

## Adopting an Existing Deployment

If the agent is already running on the spoke cluster (installed via `helm install` or manually), you can adopt it into ArgoCD without redeploying. The key difference from a fresh install is that the namespace and secrets already exist — you just need to make `values-override.yaml` match the current state so ArgoCD's first sync is a no-op.

### Step 1: Prepare the Spoke Cluster

The ArgoCD ServiceAccount is still needed. Log in to the spoke and create it:

```bash
oc login $OCP_SPOKE_SERVER

oc create namespace rh-lightspeed-agent-argocd
oc create sa rh-lightspeed-agent-argocd-sa -n rh-lightspeed-agent-argocd
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:rh-lightspeed-agent-argocd:rh-lightspeed-agent-argocd-sa

oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argocd-admin-sa-secret
  namespace: rh-lightspeed-agent-argocd
  annotations:
    kubernetes.io/service-account.name: rh-lightspeed-agent-argocd-sa
type: kubernetes.io/service-account-token
EOF

TOKEN=$(oc get secret argocd-admin-sa-secret -n rh-lightspeed-agent-argocd \
  -o jsonpath='{.data.token}' | base64 -d)
CA=$(oc get secret argocd-admin-sa-secret -n rh-lightspeed-agent-argocd \
  -o jsonpath='{.data.ca\.crt}' | base64)
echo "TOKEN: $TOKEN"
echo "CA: $CA"
```

No need to create the namespace or secrets — they already exist from the current deployment.

### Step 2: Register the Spoke on the Hub

Log in to the hub and register the spoke cluster:

```bash
oc login $OCP_HUB_SERVER

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
  server: $OCP_SPOKE_SERVER
  config: |
    {
      "bearerToken": "<TOKEN from step 1>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<CA from step 1>"
      }
    }
EOF
```

### Step 3: Match values-override to the Current Deployment

This is the critical step. Capture the current image tags and config from the spoke so the first ArgoCD sync doesn't change anything:

```bash
oc login $OCP_SPOKE_SERVER

# Get current image tags
oc get deployment -n rh-lightspeed-agent -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}'
```

Update `openshift/values-override.yaml` to match. Make sure the image tags, deploymentMode, providerUrl, and any other config values reflect what's currently running:

```yaml
agent:
  image:
    tag: <current agent tag>
  providerUrl: "https://<current-route-host>"

mcp:
  image:
    tag: <current mcp tag>

handler:
  image:
    tag: <current handler tag>

deploymentMode: <current mode>

secrets:
  create: false
```

### Step 4: Apply with Manual Sync First

Update `openshift/application.yaml` with the spoke destination and namespace:

```yaml
  destination:
    server: $OCP_SPOKE_SERVER
    namespace: <existing-agent-namespace>
```

Temporarily disable auto-sync to review what ArgoCD will do before it makes changes. Comment out the `automated` block:

```yaml
  syncPolicy:
    # automated:
    #   selfHeal: true
    #   prune: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
```

Commit, push, and apply:

```bash
git add -A
git commit -m "adopt existing deployment on spoke cluster"
git push origin main

oc login $OCP_HUB_SERVER
oc apply -f openshift/application.yaml
```

### Step 5: Review and Sync

Check what ArgoCD detects as differences:

```bash
# View the app status — should show OutOfSync if there are diffs
oc get application lightspeed-agent -n openshift-gitops

# Open the ArgoCD UI to inspect diffs visually
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

If the diff is clean (only metadata/annotation differences from ServerSideApply), trigger a manual sync:

```bash
oc exec -n openshift-gitops deploy/openshift-gitops-server -- \
  argocd app sync lightspeed-agent --local-only=false
```

Or sync from the ArgoCD UI.

### Step 6: Enable Auto-Sync

Once the first sync succeeds and the agent is still running correctly, re-enable auto-sync in `openshift/application.yaml`:

```yaml
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

Commit and push:

```bash
git add openshift/application.yaml
git commit -m "enable auto-sync after successful adoption"
git push origin main
oc apply -f openshift/application.yaml
```

From this point, all updates go through the GitOps PR workflow.
