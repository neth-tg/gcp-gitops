# Learning Path: ArgoCD + Twingate Kubernetes Operator

This project is a hands-on learning exercise. The goal is not just to get things running —
it's to understand **why** each piece exists and how it fits together.

Each phase has tasks for you to attempt before looking at solutions. The `.claude/skills/argocd-twingate.md`
skill gives Claude context to guide you through each step as a tutor, not just an answer machine.

---

## What You're Building

A GitOps-managed GKE cluster that:
1. Runs ArgoCD as the continuous delivery engine
2. Uses ArgoCD to deploy the Twingate Kubernetes Operator
3. Uses the operator to expose services and the Kubernetes API via Twingate
4. Eventually: configures the Twingate Identity Firewall for proxied, identity-aware access

```
GitHub (gcp-gitops repo)
        │
        │  ArgoCD watches this repo
        ▼
   ArgoCD (in GKE)
        │
        ├── deploys Twingate Operator (via Helm)
        ├── deploys TwingateConnector CRs
        └── deploys TwingateResource + TwingateResourceAccess CRs
                          │
                          ▼
                 Twingate Control Plane
                          │
                          ▼
              Your Twingate Client (laptop)
```

---

## Infrastructure

**GCP Project:** `twingate-playground`
**VPC:** `neth-argo-vpc` (custom, us-central1)

| Subnet | CIDR | Purpose |
|--------|------|---------|
| `neth-argo-cluster` | `10.0.0.0/24` | GKE cluster nodes |
| `neth-argo-connectors` | `10.0.1.0/24` | Twingate connectors (future) |

> **Important:** The playground auto-deletes resources without specific labels. Always add
> the required keep/owner labels when creating resources.

---

## Repo Structure (target state)

```
gcp-gitops/
├── apps/                            # ArgoCD Application manifests (app-of-apps root)
│   ├── twingate-operator.yaml       # App → base/twingate-operator
│   ├── twingate-connectors.yaml     # App → base/twingate-connectors
│   └── twingate-resources.yaml      # App → base/twingate-resources
├── base/                            # Base Kustomize configs
│   ├── twingate-operator/           # Helm chart source + values
│   ├── twingate-connectors/         # TwingateConnector custom resources
│   └── twingate-resources/          # TwingateResource + TwingateResourceAccess CRs
└── environments/
    └── playground/                  # Kustomize overlays for this environment
```

---

## Phase 0: Provision the GKE Cluster

**Concept:** ArgoCD needs a cluster to run on — the cluster itself is a prerequisite,
not a GitOps-managed artifact. This is the "bootstrap problem": you can't use ArgoCD
to create the thing ArgoCD lives on. The cluster is provisioned imperatively (or via
Terraform in a separate infra repo). Everything *on* the cluster is managed by ArgoCD.

**Your tasks:**
- [ ] Determine what label(s) the playground auto-deletion policy requires
- [ ] Create the GKE cluster in `neth-argo-cluster` subnet, single zone `us-central1-a`
  - Use `e2-small` spot nodes (2–3 nodes, 32GB disk)
  - Enable Workload Identity (`--workload-pool`)
  - Add required labels to prevent auto-deletion
- [ ] Get cluster credentials: `gcloud container clusters get-credentials ...`
- [ ] Verify: `kubectl get nodes`

**Cost note:** Use a **zonal** cluster (not regional) to qualify for the free cluster
management tier. Regional clusters cost ~$73/month extra for no benefit in a learning setup.

---

## Phase 1: Bootstrap ArgoCD

**Concept:** ArgoCD is installed imperatively the first time. After that, it can manage
itself (and everything else) via GitOps. The install creates: CRDs for Application/
AppProject/etc., Deployments for argocd-server/repo-server/application-controller,
and RBAC resources.

**Your tasks:**
- [ ] Create the `argocd` namespace
- [ ] Apply the ArgoCD install manifest (non-HA for learning)
- [ ] Explore what was created: `kubectl get all -n argocd`, `kubectl get crd | grep argo`
- [ ] Retrieve the initial admin password (hint: it's in a Secret)
- [ ] Port-forward the argocd-server and log in to the web UI
- [ ] Install the `argocd` CLI and log in via CLI
- [ ] Add your GitHub repo as a source in ArgoCD

**Questions to answer as you go:**
- What's the difference between argocd-server, argocd-repo-server, and argocd-application-controller?
- Why is the initial password stored in a Secret rather than shown in the install output?

**Key docs:**
- https://argo-cd.readthedocs.io/en/stable/getting_started/

---

## Phase 2: Your First ArgoCD Application

**Concept:** An ArgoCD `Application` is a Kubernetes CR that links a Git source
(repo + path + revision) to a destination (cluster + namespace). ArgoCD continuously
reconciles the live cluster state toward what's in Git.

**Your tasks:**
- [ ] Write an `Application` manifest by hand for the ArgoCD guestbook example:
  - Source: `https://github.com/argoproj/argocd-example-apps`, path: `guestbook`
  - Destination: your cluster, namespace: `guestbook`
- [ ] Apply it and watch it sync in the UI
- [ ] Make a manual change to the deployed resource (`kubectl edit`) and observe drift detection
- [ ] Delete the Application and observe what happens to the deployed resources (hint: depends on finalizers)

**Questions to answer:**
- What does `syncPolicy.automated` with `selfHeal: true` do differently?
- What is the difference between `Synced` and `Healthy` status?

---

## Phase 3: App-of-Apps Pattern

**Concept:** The app-of-apps pattern uses one "root" ArgoCD Application whose Git source
is a directory of other Application manifests. ArgoCD deploys the root app, which causes
it to discover and deploy all the child apps. This is how you manage many applications
from a single GitOps repo without manually applying each Application CR.

**Your tasks:**
- [ ] Write a root Application CR that points ArgoCD at your `apps/` directory in this repo
- [ ] Write a stub `apps/twingate-operator.yaml` Application pointing at `base/twingate-operator/`
  (the directory doesn't need to exist yet — observe what ArgoCD does with a missing path)
- [ ] Push both to GitHub, apply the root Application, watch ArgoCD discover the child app
- [ ] Create `base/twingate-operator/` with a placeholder `kustomization.yaml`

**Questions to answer:**
- What happens to child apps if you delete the root app?
- How would you add a new application to your cluster in the future using this pattern?

---

## Phase 4: Deploy Twingate Operator via ArgoCD

**Concept:** ArgoCD can deploy Helm charts directly from an OCI or HTTP Helm repository.
You define the chart source, version, and values inside the Application CR. ArgoCD renders
the chart and applies the resulting manifests — you never run `helm install` manually.

**Prerequisites (Twingate side):**
- Twingate account with admin access
- A Remote Network configured for your Kubernetes cluster
- An API token with Read/Write/Provision permissions
- Note the Remote Network ID

**Your tasks:**
- [ ] Create a Kubernetes Secret for the Twingate API token (manually for now)
- [ ] Write `apps/twingate-operator.yaml` as an ArgoCD Application using Helm source:
  - Chart: `oci://ghcr.io/twingate/helmcharts/twingate-operator`
  - Values: reference your API token secret, set your network slug
- [ ] Push and watch ArgoCD deploy the operator
- [ ] Verify CRDs were registered: `kubectl get crd | grep twingate`
- [ ] Verify the operator pod is running: `kubectl get pods -n <operator-namespace>`

**Operator docs:** https://github.com/Twingate/kubernetes-operator/wiki/Getting-Started

---

## Phase 5: Deploy a Connector and Expose a Service

**Concept:** `TwingateConnector` tells the operator to deploy a connector pod in the cluster.
`TwingateResource` exposes a Kubernetes service as a resource in your Twingate network.
`TwingateResourceAccess` defines which users/groups can access that resource — this is your
zero-trust access policy as code.

**Your tasks:**
- [ ] Write a `TwingateConnector` CR in `base/twingate-connectors/`
- [ ] Deploy a simple test service (nginx) in the cluster
- [ ] Write a `TwingateResource` CR to expose the nginx service
- [ ] Write a `TwingateResourceAccess` CR granting yourself access
- [ ] Verify connectivity: reach the service via your Twingate client
- [ ] Commit everything to Git and verify ArgoCD deploys it (not manual kubectl)

---

## Phase 6: Identity Firewall — Kubernetes API Access

**Concept:** The Twingate Identity Firewall is a Layer 7 reverse proxy (the "Twingate Gateway")
deployed in your environment. Instead of connecting your Twingate client directly to a port,
traffic flows through the Gateway, which authenticates your identity with Twingate and forwards
it to the backend. For Kubernetes: your identity is forwarded to the kube-apiserver, enabling
RBAC rules keyed to your Twingate user.

Currently supported protocols: **Kubernetes API**
Coming soon: SSH, HTTPS, databases, MCP

**Your tasks:**
- [ ] Deploy the Twingate Kubernetes Access Gateway
- [ ] Configure a TwingateResource pointing at the kube-apiserver
- [ ] Set up RBAC (ClusterRoleBinding or RoleBinding) tied to your Twingate identity
- [ ] Test: access the cluster via `kubectl` routed through Twingate (not direct kubeconfig)
- [ ] Inspect the audit trail in the Twingate admin dashboard

**Docs:**
- https://www.twingate.com/docs/identity-firewall
- https://www.twingate.com/docs/kubernetes-access
- https://github.com/Twingate/kubernetes-access-gateway

---

## Phase 7: SSH Privileged Access (future)

SSH privileged access via the Identity Firewall is in development. As Twingate staff you may
get early access before public release.

The pattern will follow the same model as Kubernetes API access:
- Deploy a Gateway alongside your SSH target
- Define a TwingateResource for the SSH service
- Identity is forwarded and policies enforced at the Gateway layer
- Full session audit trail

Watch internal channels for early access availability.

---

## Key Concepts Reference

| Concept | What it is |
|---------|-----------|
| ArgoCD Application | CR linking a Git source to a cluster destination |
| App-of-apps | Root Application whose source is a dir of other Application manifests |
| Sync policy | Manual vs auto; `selfHeal` re-applies drift; `prune` removes deleted resources |
| TwingateConnector | Deploys a connector pod inside the cluster |
| TwingateResource | Exposes a K8s service as a Twingate resource |
| TwingateResourceAccess | Zero-trust access policy: who can reach a TwingateResource |
| Identity Firewall | Layer 7 proxy that authenticates + forwards user identity to backends |
| Workload Identity | GKE feature: K8s ServiceAccounts map to GCP IAM — no key files |

## Useful Commands

```bash
# Cluster
gcloud container clusters get-credentials neth-argo-cluster --zone=us-central1-a --project=twingate-playground
kubectl get nodes

# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ArgoCD CLI
argocd login localhost:8080
argocd app list
argocd app sync <app-name>
argocd app diff <app-name>

# Twingate operator CRDs (after install)
kubectl get crd | grep twingate
kubectl get twingateconnectors -A
kubectl get twingateresources -A
kubectl get twingateresourceaccesses -A
```
