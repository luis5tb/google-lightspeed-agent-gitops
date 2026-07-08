# Lightspeed Agent GitOps

ArgoCD deployment configuration for the [Red Hat Lightspeed Agent](https://github.com/RHEcosystemAppEng/google-lightspeed-agent). This repo controls **when and where** the agent is deployed. The Helm charts and application code live in the app repo.

## Structure

```
openshift/
├── application.yaml        # ArgoCD Application CR — syncs the OpenShift Helm chart
├── values-override.yaml    # Environment-specific overrides (image tags, config)
└── README.md               # OpenShift deployment guide
google-cloud/
├── application.yaml        # ArgoCD Application CR — triggers Cloud Build for Cloud Run
├── values-override.yaml    # Environment-specific overrides (project ID, image tags)
└── README.md               # Google Cloud deployment guide
```

## How It Works

Each Application CR uses ArgoCD multi-source to combine:
- **This repo** — environment overrides (`values-override.yaml`)
- **App repo** — Helm charts and templates (`deploy/openshift/` or `deploy/gitops/google-cloud/`)

ArgoCD watches both repos. When a change is merged to `main` in either repo, ArgoCD re-renders the Helm chart and syncs.

## Deployment Targets

| Target | What ArgoCD manages | Where the workload runs | Guide |
|---|---|---|---|
| **OpenShift** | K8s resources directly (Deployments, Services, Routes) | OpenShift cluster | [openshift/README.md](openshift/README.md) |
| **Google Cloud** | Intermediate K8s resources (ConfigMap, Secret, Job) | Cloud Run on GCP | [google-cloud/README.md](google-cloud/README.md) |

## Common Setup (Cluster-Admin)

These steps are shared across both targets and require cluster-admin access on the OpenShift cluster where ArgoCD runs.

### 1. Install the OpenShift GitOps operator (if not already installed)

```bash
bash deploy/gitops/setup/install-gitops-operator.sh
```

See [setup docs](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/tree/main/deploy/gitops/setup) in the app repo.

### 2. Create the team's namespace for Application CRs

```bash
oc create namespace rh-lightspeed-agent-argocd
```

### 3. Configure ArgoCD to watch the Application CR namespace

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge -p '
spec:
  sourceNamespaces:
    - openshift-gitops
    - rh-lightspeed-agent-argocd
'
```

### 4. Create an AppProject to scope what the team can deploy

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

### 5. Grant team access in ArgoCD RBAC

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

### Google Cloud target: additional prerequisites

The Google Cloud target requires a GCP service account. See [google-cloud/README.md](google-cloud/README.md#prerequisites) for details.
