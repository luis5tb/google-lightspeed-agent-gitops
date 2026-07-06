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

For deploying to a remote cluster, see the [cross-cluster docs](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/tree/main/deploy/gitops/README.md#cross-cluster-deployment) in the app repo. The key change is updating `destination.server` in the Application CR.
