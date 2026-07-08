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

Setup is split between a **cluster-admin** (one-time) and the **team** (ongoing).

### Cluster-Admin: One-Time Setup

These steps require cluster-admin access. They configure the existing ArgoCD instance in `openshift-gitops` to support the team's namespace and grant the team visibility.

**1. Install the OpenShift GitOps operator** (if not already installed):

```bash
bash deploy/gitops/setup/install-gitops-operator.sh
```

See [setup docs](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/tree/main/deploy/gitops/setup) in the app repo. For the Google Cloud target, also run `install-eso-operator.sh` and `setup-gcp-sa.sh`.

**2. Create the team's namespace for Application CRs:**

```bash
oc create namespace rh-lightspeed-agent-argocd
```

**3. Grant ArgoCD access to the agent namespace:**

The ArgoCD controller needs permissions to manage resources in the namespace where the agent runs:

```bash
oc label namespace rh-lightspeed-agent \
  argocd.argoproj.io/managed-by=openshift-gitops

oc create rolebinding argocd-admin \
  --clusterrole=admin \
  --serviceaccount=openshift-gitops:openshift-gitops-argocd-application-controller \
  -n rh-lightspeed-agent
```

**4. Configure ArgoCD to watch the Application CR namespace:**

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge -p '
spec:
  sourceNamespaces:
    - openshift-gitops
    - rh-lightspeed-agent-argocd
'
```

**5. Create an AppProject** to scope what the team can deploy:

```bash
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: rh-lightspeed-agent
  namespace: openshift-gitops
spec:
  description: Red Hat Lightspeed Agent project
  sourceRepos:
    - 'https://github.com/RHEcosystemAppEng/google-lightspeed-agent.git'
    - 'https://github.com/RHEcosystemAppEng/google-lightspeed-agent-gitops.git'
  sourceNamespaces:
    - rh-lightspeed-agent-argocd
  destinations:
    - namespace: rh-lightspeed-agent
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
EOF
```

> Update `sourceRepos` if using fork URLs for testing. Add more `destinations` for cross-cluster deployments.

**6. Grant team access in ArgoCD RBAC:**

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge -p '
spec:
  rbac:
    policy: |
      g, system:cluster-admins, role:admin
      g, cluster-admins, role:admin
      g, admins, role:admin
'
```

### Team: Deploy

Once the cluster-admin has completed the steps above, team members can manage deployments without cluster-admin access.

**1. Edit `openshift/application.yaml`** — set the team namespace and project:

```yaml
metadata:
  name: lightspeed-agent
  namespace: rh-lightspeed-agent-argocd    # team namespace, NOT openshift-gitops
spec:
  project: rh-lightspeed-agent             # AppProject, NOT default
  destination:
    server: https://kubernetes.default.svc
    namespace: rh-lightspeed-agent
```

**2. Edit `openshift/values-override.yaml`** with environment values (image tags, providerUrl, etc.).

**3. Commit, push, and apply:**

```bash
git add -A
git commit -m "configure deployment"
git push origin main
oc apply -f openshift/application.yaml
```

**4. Access the ArgoCD UI:**

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
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
  # When using Vertex AI with a pre-existing GCP SA key secret:
  # Set to any non-empty value to enable the volume mount and
  # GOOGLE_APPLICATION_CREDENTIALS env var in the agent pod.
  # With create: false, this only enables the mount — it does NOT
  # create the secret. The secret must already exist as
  # <release-name>-gcp-sa-key in the namespace.
  # With create: true, set this to the actual JSON key content.
  gcpServiceAccountKey: "true"
```

**GCP SA key secret (Vertex AI only):** If using Vertex AI (`google.useVertexAI: true`), the agent needs a GCP service account key. Create the secret with the key named `sa-key.json`:

```bash
oc create secret generic lightspeed-agent-gcp-sa-key \
  --from-file=sa-key.json=<path-to-your-gcp-sa-key.json> \
  -n ${AGENT_NAMESPACE}
```

The key name **must** be `sa-key.json` — the chart mounts it at `/var/run/secrets/gcp/sa-key.json`. If you have an existing key file with a different name (e.g., `credentials.json`), use `--from-file=sa-key.json=credentials.json` to rename it during creation.

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
