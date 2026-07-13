# Google Cloud Deployment

ArgoCD-managed deployment of the Lightspeed Agent on Google Cloud (Cloud Run). Unlike the OpenShift target where ArgoCD manages workloads directly, here ArgoCD manages **intermediate K8s resources on OpenShift** and a PostSync hook Job triggers Cloud Build to deploy Cloud Run services on GCP.

> **Prerequisite:** Complete the [common cluster-admin setup](../README.md#common-setup-cluster-admin) first.

## How It Works

```
PR merged → ArgoCD syncs K8s resources on OpenShift → PostSync Job → gcloud builds submit → Cloud Build → Cloud Run
```

1. A PR changes image tags or config in `values-override.yaml` in this repo
2. PR merges to `main`
3. ArgoCD detects the change and begins sync
4. ArgoCD creates/updates these K8s resources on the OpenShift cluster:
   - **ConfigMap** — maps `values.yaml` fields to Cloud Build `_VARIABLE` substitutions
   - **Secret** — the GCP SA key from the bootstrap secret (`gcp-sa-bootstrap`), used by the deploy Job
   - **ServiceAccount** + RBAC — grants the deploy Job read access to secrets and configmaps
5. **PostSync hook Job** — clones the app repo and runs `gcloud builds submit` with substitutions from the ConfigMap
6. **Cloud Build** — pulls images from Quay.io, scans with Trivy, pushes to GCR, deploys Cloud Run services

The workload runs on GCP. The OpenShift cluster only hosts the ArgoCD orchestration resources.

## Prerequisites

In addition to the [common setup](../README.md#common-setup-cluster-admin), the Google Cloud target requires:

### 1. Create the GCP service account and bootstrap secret

```bash
export GOOGLE_CLOUD_PROJECT=my-project-id
bash deploy/gitops/setup/setup-gcp-sa.sh
```

If your Cloud Run runtime SA uses a non-default name (the default is `lightspeed-agent`), set `CLOUD_RUN_SA` to match:

```bash
export GOOGLE_CLOUD_PROJECT=my-project-id
CLOUD_RUN_SA=sa-lightspeed-agent bash deploy/gitops/setup/setup-gcp-sa.sh
```

This creates a GCP service account (`lightspeed-gitops`) with the required IAM roles and stores its key in a K8s Secret (`gcp-sa-bootstrap`) on the OpenShift cluster, used by the deploy Job.

| Role | Scope | Purpose |
|---|---|---|
| `roles/cloudbuild.builds.editor` | Project | Submit Cloud Build pipelines |
| `roles/run.admin` | Project | Deploy Cloud Run services |
| `roles/serviceusage.serviceUsageConsumer` | Project | `gcloud builds submit` API access |
| `roles/iam.serviceAccountUser` | Cloud Run runtime SA | Impersonate the runtime SA |

### 2. Grant ArgoCD access to the namespace

If not already done for the OpenShift target:

```bash
oc label namespace rh-lightspeed-agent \
  argocd.argoproj.io/managed-by=openshift-gitops

oc create rolebinding argocd-admin \
  --clusterrole=admin \
  --serviceaccount=openshift-gitops:openshift-gitops-argocd-application-controller \
  -n rh-lightspeed-agent
```

Both the OpenShift and Google Cloud targets share the same namespace — these commands are the same as in the [OpenShift setup](../openshift/README.md#1-grant-argocd-access-to-the-agent-namespace).

## Deploy

> **New environment?** If the GCP infrastructure doesn't exist yet (Cloud SQL, Redis, Pub/Sub, service accounts, secrets in GCP Secret Manager), run the one-time setup first. See the [Cloud Run deployment guide](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/tree/main/deploy/cloudrun/README.md) in the app repo for details. This is not needed when [adopting an existing deployment](#adopting-an-existing-gcp-environment).

### 1. Edit `values-override.yaml`

Set your GCP project ID, image tags, and any environment-specific configuration:

```yaml
project:
  id: my-gcp-project-id

images:
  agent:
    tag: v1.2.3
  handler:
    tag: v1.2.3
  mcp:
    tag: "20260701"

services:
  serviceAccountName: lightspeed-agent  # match your Cloud Run runtime SA

# Pub/Sub — Marketplace uses a cross-project topic
pubsub:
  topic: projects/cloudcommerceproc-prod/topics/my-gcp-project-id
  subscription: marketplace-events-sub  # match existing subscription name

# Load balancer — set domains when GCLB is enabled
loadBalancer:
  agent:
    enabled: "true"
    domain: agent.example.com
  handler:
    enabled: "true"
    domain: dcr.example.com

deploy:
  gitRepo: https://github.com/RHEcosystemAppEng/google-lightspeed-agent.git
  gitBranch: main
```

> **Why `deploy.gitRepo` and `deploy.gitBranch`?** ArgoCD clones the repo to render the Helm chart (using `application.yaml`'s `repoURL` + `targetRevision`), but the PostSync deploy Job runs in a separate container and does its own `git clone` to get `cloudbuild.yaml` and the Cloud Run service YAMLs. These two clones are independent — the Job has no access to ArgoCD's checkout. `deploy.gitRepo` and `deploy.gitBranch` control the Job's clone and should match the `application.yaml` source. If using a fork, set `deploy.gitRepo` to the fork URL.

See `deploy/gitops/google-cloud/values.yaml` in the [app repo](https://github.com/RHEcosystemAppEng/google-lightspeed-agent) for all available parameters and defaults.

### 2. Apply the Application CR

```bash
git add -A
git commit -m "configure GCP deployment"
git push origin main
oc apply -f google-cloud/application.yaml
```

ArgoCD will create the K8s resources and the PostSync Job will trigger Cloud Build.

### 3. Monitor

```bash
# ArgoCD sync status
oc get application lightspeed-agent-gcp -n rh-lightspeed-agent-argocd \
  -o jsonpath='{.status.sync.status}{" "}{.status.health.status}'

# PostSync Job status
oc get jobs -n rh-lightspeed-agent -l app.kubernetes.io/component=deploy

# Cloud Build logs (on GCP)
gcloud builds list --project=my-gcp-project-id --limit=5
```

### 4. (Optional) Private git repositories

If the app repo is private, create a token secret:

```bash
oc create secret generic git-credentials \
  --from-literal=token=<GITHUB_PAT> \
  -n rh-lightspeed-agent
```

Then set `deploy.gitTokenSecret: git-credentials` in your `values-override.yaml`.

## Adopting an Existing GCP Environment

If the agent is already running on Cloud Run (deployed via `deploy.sh`, `deploy-cloudbuild.sh`, or manually), you can bring it under ArgoCD/GitOps management. Unlike OpenShift adoption, ArgoCD doesn't take over existing K8s resources — instead, it creates new intermediate resources on OpenShift and the first sync triggers Cloud Build, which redeploys the Cloud Run services.

The key is making `values-override.yaml` match the current GCP state so Cloud Build deploys the same configuration that's already running.

### Step 1: Capture the current GCP state

```bash
export GOOGLE_CLOUD_PROJECT=my-gcp-project-id
export GOOGLE_CLOUD_LOCATION=us-central1

# Get current Cloud Run service details
gcloud run services describe lightspeed-agent \
  --region=${GOOGLE_CLOUD_LOCATION} --project=${GOOGLE_CLOUD_PROJECT} \
  --format='value(spec.template.spec.containers[0].image)'

gcloud run services describe marketplace-handler \
  --region=${GOOGLE_CLOUD_LOCATION} --project=${GOOGLE_CLOUD_PROJECT} \
  --format='value(spec.template.spec.containers[0].image)'

# Check load balancer configuration
gcloud compute forwarding-rules list --project=${GOOGLE_CLOUD_PROJECT}
gcloud compute ssl-certificates list --project=${GOOGLE_CLOUD_PROJECT}

# Check Pub/Sub (Marketplace uses a cross-project topic)
gcloud pubsub subscriptions list --project=${GOOGLE_CLOUD_PROJECT}

# Check Cloud Run service account name
gcloud run services describe lightspeed-agent \
  --region=${GOOGLE_CLOUD_LOCATION} --project=${GOOGLE_CLOUD_PROJECT} \
  --format='value(spec.template.spec.serviceAccountName)'
```

### Step 2: Match `values-override.yaml` to current state

Update `google-cloud/values-override.yaml` with the values from step 1:

```yaml
project:
  id: my-gcp-project-id

images:
  agent:
    tag: <current agent image tag>
  handler:
    tag: <current handler image tag>
  mcp:
    tag: <current mcp image tag>

loadBalancer:
  agent:
    enabled: "true"              # match current LB state
    domain: agent.example.com    # match current domain
  handler:
    enabled: "true"
    domain: dcr.example.com

services:
  serviceAccountName: lightspeed-agent  # must match the Cloud Run runtime SA

# Pub/Sub — Marketplace uses a cross-project topic
pubsub:
  topic: projects/cloudcommerceproc-prod/topics/my-project-id
  subscription: marketplace-events-sub  # match existing subscription name

deploy:
  gitRepo: https://github.com/RHEcosystemAppEng/google-lightspeed-agent.git
  gitBranch: main                # must match application.yaml targetRevision
```

> **`services.serviceAccountName`** must match the Cloud Run runtime service account name used by `deploy/cloudrun/setup.sh` (`SERVICE_ACCOUNT_NAME`, default: `lightspeed-agent`). If your environment uses a different name (e.g., `sa-lightspeed-agent`), set it here — a mismatch causes `iam.serviceaccounts.actAs` permission errors during Cloud Build deployment.

> **`pubsub.topic`** — Google Cloud Marketplace provisions a cross-project topic (e.g., `projects/cloudcommerceproc-prod/topics/{project-id}`). The default `marketplace-entitlements` is for local development. Check your existing subscriptions with `gcloud pubsub subscriptions list` and set `pubsub.subscription` to the existing subscription name to avoid creating a duplicate.

### Step 3: Ensure prerequisites are in place

Complete the [prerequisites](#prerequisites) above (GCP SA bootstrap secret, namespace labeling). These create new resources on OpenShift that don't conflict with the existing GCP deployment.

### Step 4: Apply the Application CR

```bash
git add -A
git commit -m "adopt existing GCP deployment"
git push origin main
oc apply -f google-cloud/application.yaml
```

ArgoCD will:
1. Create the K8s resources on OpenShift (ConfigMap, Secret, ServiceAccount)
2. Run the PostSync Job, which triggers Cloud Build
3. Cloud Build will redeploy Cloud Run with the values from the ConfigMap

Since the values match the current deployment, this is a **redeploy-in-place** — the same images and configuration are deployed over the existing services. Cloud Run handles this as a no-op revision if nothing changed.

### Step 5: Verify

```bash
# ArgoCD status
oc get application lightspeed-agent-gcp -n rh-lightspeed-agent-argocd \
  -o jsonpath='{.status.sync.status}{" "}{.status.health.status}'

# Cloud Run services are still healthy
gcloud run services describe lightspeed-agent \
  --region=${GOOGLE_CLOUD_LOCATION} --project=${GOOGLE_CLOUD_PROJECT} \
  --format='value(status.conditions[0].status)'

gcloud run services describe marketplace-handler \
  --region=${GOOGLE_CLOUD_LOCATION} --project=${GOOGLE_CLOUD_PROJECT} \
  --format='value(status.conditions[0].status)'
```

From this point, all updates go through the GitOps PR workflow.

## Day-2: Updating Image Tags

```bash
git checkout -b bump-agent-v1.3.0
vi google-cloud/values-override.yaml       # change image tags
git add google-cloud/values-override.yaml
git commit -m "chore: bump agent to v1.3.0"
git push origin bump-agent-v1.3.0
gh pr create --title "Bump agent to v1.3.0"
# SRE reviews and merges → ArgoCD syncs ConfigMap → PostSync Job → Cloud Build → Cloud Run
```

## Multiple Instances on the Same GCP Project

To deploy multiple agent instances to the same GCP project (e.g., staging + production), each instance needs unique GCP resource names, its own Pub/Sub subscription, and a separate directory in this repo. See the [multi-instance deployment guide](https://github.com/RHEcosystemAppEng/google-lightspeed-agent/blob/main/deploy/gitops/README.md#multiple-instances-on-the-same-gcp-project) in the app repo for the full step-by-step, including:

- Which GCP resources collide and their override keys
- GCP resource name length limits
- Pub/Sub event routing via `serviceControlServiceName`
- Redis rate limit isolation via `rateLimit.keyPrefix`
- Per-instance directory structure for this repo
- Same-namespace vs separate-namespace options

## Cloud Build Substitutions

The ConfigMap maps `values.yaml` fields to Cloud Build `_VARIABLE` names. All substitutions match `cloudbuild.yaml` in the app repo root. Key mappings:

| values.yaml | Cloud Build Variable |
|---|---|
| `project.id` | `GOOGLE_CLOUD_PROJECT` |
| `project.region` | `_REGION` |
| `images.agent.tag` | `_IMAGE_TAG` |
| `images.agent.source` | `_AGENT_SOURCE_IMAGE` |
| `services.agent.name` | `_SERVICE_NAME` |
| `loadBalancer.agent.enabled` | `_ENABLE_LB_AGENT` |
| `security.scanSeverity` | `_SCAN_SEVERITY` |

See `deploy/gitops/google-cloud/templates/deployment-config.yaml` in the app repo for the complete mapping.
