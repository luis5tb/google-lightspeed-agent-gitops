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

Log in to the spoke cluster and create a ServiceAccount with namespace-scoped permissions. No cluster-admin required — ArgoCD only needs access to the agent namespace:

```bash
oc login $OCP_SPOKE_SERVER

export AGENT_NAMESPACE=lightspeed-agent   # adjust to your namespace

oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rh-lightspeed-agent-argocd-sa
  namespace: ${AGENT_NAMESPACE}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-manager
  namespace: ${AGENT_NAMESPACE}
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets",
                "serviceaccounts", "persistentvolumeclaims",
                "endpoints", "events"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Uncomment if you have monitoring.coreos.com permissions and want ServiceMonitors:
  # - apiGroups: ["monitoring.coreos.com"]
  #   resources: ["servicemonitors"]
  #   verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-manager
  namespace: ${AGENT_NAMESPACE}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-manager
subjects:
  - kind: ServiceAccount
    name: rh-lightspeed-agent-argocd-sa
    namespace: ${AGENT_NAMESPACE}
---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-sa-secret
  namespace: ${AGENT_NAMESPACE}
  annotations:
    kubernetes.io/service-account.name: rh-lightspeed-agent-argocd-sa
type: kubernetes.io/service-account-token
EOF

# Save these values — you'll need them on the hub
TOKEN=$(oc get secret argocd-sa-secret -n ${AGENT_NAMESPACE} \
  -o jsonpath='{.data.token}' | base64 -d)
CA=$(oc get secret argocd-sa-secret -n ${AGENT_NAMESPACE} \
  -o jsonpath='{.data.ca\.crt}' | base64)
echo "TOKEN: $TOKEN"
echo "CA: $CA"
```

Create the app secrets on the spoke cluster:

```bash
cp openshift/secrets.yaml.example my-secrets.yaml
vi my-secrets.yaml   # fill in real credentials
oc apply -f my-secrets.yaml -n ${AGENT_NAMESPACE}
```

See the [OpenShift deployment docs](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/blob/main/deploy/openshift/README.md#4-configure-values) in the app repo for details on each secret field and configuration options (Vertex AI, persistent sessions, etc.).

> **Note:** This manual approach is for testing and development. For production, use a secrets management solution such as Vault, External Secrets Operator, or SealedSecrets.

### Step 2: Register the Spoke on the Hub Cluster

Log in to the hub cluster and create the ArgoCD cluster connection Secret. The `namespaces` field limits ArgoCD to the agent namespace only:

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
  namespaces: "${AGENT_NAMESPACE}"
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

If the agent is already running (installed via `helm install` or manually), you can bring it under ArgoCD management without redeploying. The key is making `values-override.yaml` match the current state so ArgoCD's first sync is a no-op.

### Single-Cluster (ArgoCD and Agent on the Same Cluster)

The simplest case — ArgoCD deploys to `https://kubernetes.default.svc` (the local cluster). No cluster registration or extra ServiceAccounts needed.

#### Step 1: Match values-override to the Current Deployment

Capture the current image tags and config so the first sync doesn't change anything:

```bash
export AGENT_NAMESPACE=rh-lightspeed-agent   # your existing agent namespace

# Get current image tags
oc get deployment -n ${AGENT_NAMESPACE} -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}'

# Get current route host
oc get route -n ${AGENT_NAMESPACE} -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.host}{"\n"}{end}'
```

Update `openshift/values-override.yaml` to match what's currently running:

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

#### Step 2: Apply with Manual Sync First

Update `openshift/application.yaml` — set the namespace to your existing one:

```yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: <existing-agent-namespace>
```

Temporarily disable auto-sync to review what ArgoCD will do before it makes changes. Comment out the `automated` block:

```yaml
  syncPolicy:
    # automated:
    #   selfHeal: true
    #   prune: true
    syncOptions:
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
```

Commit, push, and apply:

```bash
git add -A
git commit -m "adopt existing single-cluster deployment"
git push origin main

oc apply -f openshift/application.yaml
```

#### Step 3: Review and Sync

Check what ArgoCD detects as differences:

```bash
# View the app status
oc get application lightspeed-agent -n openshift-gitops

# Open the ArgoCD UI to inspect diffs visually
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

If the diff is clean (only metadata/annotation differences from ServerSideApply), trigger a manual sync from the ArgoCD UI or CLI.

#### Step 4: Enable Auto-Sync

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

### Cross-Cluster (ArgoCD on Hub, Agent on Spoke)

Same approach as above, but with extra steps to register the spoke cluster and create a ServiceAccount for ArgoCD.

#### Step 1: Create ArgoCD ServiceAccount on the Spoke

No need to create the namespace or secrets — they already exist. Just the SA with namespace-scoped permissions:

```bash
oc login $OCP_SPOKE_SERVER

export AGENT_NAMESPACE=rh-lightspeed-agent   # your existing agent namespace

oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rh-lightspeed-agent-argocd-sa
  namespace: ${AGENT_NAMESPACE}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-manager
  namespace: ${AGENT_NAMESPACE}
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets",
                "serviceaccounts", "persistentvolumeclaims",
                "endpoints", "events"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Uncomment if you have monitoring.coreos.com permissions and want ServiceMonitors:
  # - apiGroups: ["monitoring.coreos.com"]
  #   resources: ["servicemonitors"]
  #   verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-manager
  namespace: ${AGENT_NAMESPACE}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-manager
subjects:
  - kind: ServiceAccount
    name: rh-lightspeed-agent-argocd-sa
    namespace: ${AGENT_NAMESPACE}
---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-sa-secret
  namespace: ${AGENT_NAMESPACE}
  annotations:
    kubernetes.io/service-account.name: rh-lightspeed-agent-argocd-sa
type: kubernetes.io/service-account-token
EOF

TOKEN=$(oc get secret argocd-sa-secret -n ${AGENT_NAMESPACE} \
  -o jsonpath='{.data.token}' | base64 -d)
CA=$(oc get secret argocd-sa-secret -n ${AGENT_NAMESPACE} \
  -o jsonpath='{.data.ca\.crt}' | base64)
echo "TOKEN: $TOKEN"
echo "CA: $CA"
```

#### Step 2: Register the Spoke on the Hub

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
  namespaces: "${AGENT_NAMESPACE}"
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

#### Step 3: Match, Apply, Review, Enable

Follow the same steps as the single-cluster adoption above (Step 1 through Step 4), but set `destination.server` to `$OCP_SPOKE_SERVER` instead of `https://kubernetes.default.svc` in `openshift/application.yaml`.
