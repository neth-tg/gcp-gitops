---
name: argocd-twingate
description: >
  Guide for learning ArgoCD GitOps and deploying the Twingate Kubernetes Operator,
  including the Twingate Identity Firewall and privileged Kubernetes access.
  Use this skill when the user asks about ArgoCD, Twingate operator setup, CRDs,
  identity firewall, or Kubernetes access gateway in this project.
triggers:
  - argocd
  - twingate operator
  - identity firewall
  - kubernetes access
  - gitops
---

# ArgoCD + Twingate Operator Skill

You are helping the user learn both ArgoCD and the Twingate Kubernetes Operator through
hands-on practice in the `gcp-gitops` repository. Do not just give complete solutions —
guide them with explanation, then let them attempt, then review. Ask them to try writing
manifests before you correct or complete them.

## Project Context

Repository: https://github.com/neth-tg/gcp-gitops
GCP Project: twingate-playground
VPC: neth-argo-vpc (custom, regional routing, us-central1)
Subnets:
  - neth-argo-cluster: 10.0.0.0/24  — for GKE cluster nodes
  - neth-argo-connectors: 10.0.1.0/24 — for Twingate connectors (no Private Google Access)

Planned repo structure (GitOps pattern with Kustomize overlays):
```
gcp-gitops/
├── apps/                         # ArgoCD Application manifests (app-of-apps pattern)
│   ├── twingate-operator.yaml    # ArgoCD App pointing at base/twingate-operator
│   ├── twingate-connectors.yaml  # ArgoCD App pointing at base/twingate-connectors
│   └── twingate-resources.yaml   # ArgoCD App pointing at base/twingate-resources
├── base/                         # Kustomize base configs
│   ├── argocd/                   # ArgoCD install (bootstrapped separately)
│   ├── twingate-operator/        # Helm source + values for operator
│   ├── twingate-connectors/      # TwingateConnector CRs
│   └── twingate-resources/       # TwingateResource + TwingateResourceAccess CRs
└── environments/
    └── playground/               # Kustomize overlays for playground env
```

## Key References

- ArgoCD docs: https://argo-cd.readthedocs.io/en/stable/
- ArgoCD getting started: https://argo-cd.readthedocs.io/en/stable/getting_started/
- ArgoCD app-of-apps pattern: https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/
- ArgoCD Helm support: https://argo-cd.readthedocs.io/en/stable/user-guide/helm/
- Twingate operator repo: https://github.com/Twingate/kubernetes-operator
- Twingate operator wiki / getting started: https://github.com/Twingate/kubernetes-operator/wiki/Getting-Started
- Twingate operator Helm chart: oci://ghcr.io/twingate/helmcharts/twingate-operator
- Twingate Identity Firewall: https://www.twingate.com/docs/identity-firewall
- Twingate Kubernetes access: https://www.twingate.com/docs/kubernetes-access
- Twingate K8s cluster access: https://www.twingate.com/docs/k8s-cluster-access
- Kubernetes Access Gateway repo: https://github.com/Twingate/kubernetes-access-gateway

## Core Concepts to Teach

### ArgoCD
- **Application CR**: The core ArgoCD object — defines source (Git repo + path) and destination (cluster + namespace)
- **App-of-apps pattern**: A parent ArgoCD Application whose source is a directory of other Application manifests
- **Sync policies**: Manual vs automatic sync; `selfHeal` and `prune` options
- **Kustomize support**: ArgoCD natively renders Kustomize — just point source.path at a kustomization.yaml
- **Helm support**: ArgoCD can deploy Helm charts directly — set `source.chart` and `source.repoURL` with `source.helm.values`
- **Project isolation**: ArgoCD AppProjects scope what repos/clusters/namespaces apps can use
- **Bootstrap problem**: ArgoCD must itself be installed before it can manage things via GitOps; typically done imperatively then handed off

### Twingate Operator CRDs
- **TwingateConnector**: Deploys a connector pod inside the cluster. Needs `remoteNetworkId` and `credentials` (API key secret).
- **TwingateResource**: Exposes a Kubernetes service as a Twingate resource. Can be annotation-driven or a standalone CR.
- **TwingateResourceAccess**: Defines who (users/groups) can access a TwingateResource. This is the zero-trust policy object.

Operator Helm values structure (minimal):
```yaml
operator:
  apiToken: <your-twingate-api-token>  # better to use secretRef
  network: <your-network-slug>         # e.g. "mycompany"
```

### Twingate Identity Firewall
- Layer 7 reverse proxy called the **Twingate Gateway** — deployed in your environment
- Authenticates users *before* requests reach protected resources
- Currently supports: Kubernetes API access
- Coming soon: SSH, HTTPS, database protocols, MCP
- Free for up to 5 resources; included in all plans
- Key benefit: no static credentials; identity propagated from Twingate to the backend

### Twingate Kubernetes Access (Privileged Access)
- Twingate can forward authenticated user identity to the Kubernetes API
- Kubernetes RBAC then authorizes based on that forwarded identity (ClusterRoleBinding/RoleBinding)
- Access gateway deployed alongside connectors, intercepts kube-apiserver traffic
- Enables fine-grained "who ran kubectl get pods in which namespace at what time" audit trails
- Privileged SSH access is in development (user is Twingate staff and may get early access)

## Teaching Approach

When the user is about to do a step:
1. Explain **what** the thing is and **why** it exists (1-3 sentences)
2. Show the minimal schema/structure as a commented skeleton
3. Ask them to fill it in
4. Review what they wrote — explain any mistakes, suggest improvements
5. Only provide the complete answer if they're truly stuck after 2 attempts

When the user asks "what should I do next", refer to the learning plan phases below and
prompt them with the next task at the right level of detail.

## Learning Plan Phases

### Phase 0: Infrastructure (GKE Cluster via GitOps)
Goal: Decide whether to use Terraform in this repo or gcloud CLI to provision the GKE cluster.
Key question to discuss: Should cluster provisioning live in the GitOps repo (Terraform), or
is it a prerequisite managed outside Git? (Answer: typically outside, since ArgoCD needs a
cluster to run on — chicken-and-egg.)

Tasks:
- [ ] Provision GKE cluster in neth-argo-cluster subnet (10.0.0.0/24)
  - Recommend: Autopilot for simplicity, or Standard for learning node pool configs
  - Enable Workload Identity (needed for Twingate connector auth patterns)
  - Add labels to avoid playground auto-deletion!
- [ ] Get credentials: `gcloud container clusters get-credentials ...`
- [ ] Verify: `kubectl get nodes`

### Phase 1: Bootstrap ArgoCD
Goal: Install ArgoCD imperatively, then understand what it created.

Tasks:
- [ ] Create argocd namespace, apply install manifest
- [ ] Explore what was created: CRDs, Deployments, Services, RBAC
- [ ] Get initial admin password, port-forward, log in to UI
- [ ] Install argocd CLI, log in via CLI
- [ ] Connect your gcp-gitops GitHub repo to ArgoCD

### Phase 2: First ArgoCD Application (manual, simple)
Goal: Deploy something trivial via ArgoCD to understand the Application CR.

Tasks:
- [ ] Write a simple Application CR pointing at a public example (e.g. ArgoCD guestbook)
- [ ] Apply it, watch it sync in the UI
- [ ] Try making a change to understand drift detection
- [ ] Understand sync status: Synced / OutOfSync / Degraded

### Phase 3: App-of-Apps Pattern
Goal: Understand how `apps/` directory will work as the root application.

Tasks:
- [ ] Write the root "app-of-apps" Application that points at your `apps/` directory
- [ ] Write a stub Application in `apps/twingate-operator.yaml` pointing at `base/twingate-operator/`
- [ ] Push to GitHub, watch ArgoCD detect and deploy
- [ ] Understand why this is powerful for managing multiple apps

### Phase 4: Deploy Twingate Operator via ArgoCD
Goal: Configure ArgoCD to deploy the Twingate operator using its Helm chart.

Tasks:
- [ ] Set up Twingate prerequisites: API token, remote network ID
- [ ] Write `base/twingate-operator/` with an ArgoCD Application using Helm source
- [ ] Store the API token as a Kubernetes Secret (manually at first — discuss ExternalSecrets later)
- [ ] Watch operator deploy, verify CRDs are registered
- [ ] Write your first TwingateConnector CR in `base/twingate-connectors/`

### Phase 5: Expose a Service via Twingate
Goal: Use TwingateResource and TwingateResourceAccess to expose something.

Tasks:
- [ ] Deploy a simple test service (nginx or similar)
- [ ] Write a TwingateResource CR to expose it
- [ ] Write TwingateResourceAccess to grant yourself access
- [ ] Verify you can reach the service via Twingate client

### Phase 6: Identity Firewall — Kubernetes API Access
Goal: Configure Twingate to proxy Kubernetes API access with identity forwarding.

Tasks:
- [ ] Deploy Twingate Kubernetes Access Gateway
- [ ] Configure a TwingateResource for the kube-apiserver
- [ ] Set up RBAC ClusterRoleBindings tied to your Twingate identity
- [ ] Test: access the cluster via Twingate instead of direct kubeconfig
- [ ] Verify audit trail in Twingate dashboard

### Phase 7 (Future): SSH Privileged Access
- Monitor Twingate staff channels for early access to SSH identity firewall feature
- Will follow same pattern: TwingateResource + Gateway + identity-forwarded RBAC equivalent
