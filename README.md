# GitOps Assignment: Argo CD + podinfo + kube-prometheus-stack

This repository demonstrates a complete GitOps workflow using Argo CD, podinfo, and kube-prometheus-stack.

## What’s inside

- `bootstrap/` — Argo CD install values, ingress, and the root Application.
- `apps/` — podinfo Application and Git-managed values.
- `platform/` — kube-prometheus-stack Application and Helm values.
- `screenshots/` — proof screenshots for Argo CD, podinfo, and Grafana.
- `README.md` — setup, decisions, traps, and design notes.

## Starting path and why

I started with the **Argo CD Helm chart** instead of raw manifests because it made the bootstrap faster, easier to customize, and simpler to keep consistent across environments. The Helm chart also let me set `server.insecure=true` up front, which avoids the ingress redirect loop trap behind HTTP ingress.

## Bootstrap commands

### 1) Install Argo CD

```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  -f bootstrap/argocd-values.yaml
kubectl apply -f bootstrap/argocd-ingress.yaml
```

### 2) Start the app-of-apps flow

```bash
kubectl apply -f bootstrap/root-application.yaml
```

### 3) Add host entries

Add these lines to `/etc/hosts`:

```text
127.0.0.1 argocd.local
127.0.0.1 podinfo.local
127.0.0.1 grafana.local
```

## Access URLs

- Argo CD UI: [http://argocd.local](http://argocd.local)
- podinfo: [http://podinfo.local](http://podinfo.local)
- Grafana: [http://grafana.local](http://grafana.local)

Default Grafana credentials:
- Username: `admin`
- Password: `admin`

## Repo layout

### `bootstrap/`
Contains the Argo CD bootstrap resources:
- `argocd-values.yaml`
- `argocd-ingress.yaml`
- `root-application.yaml`

### `apps/`
Contains the podinfo Application and its Git-based values:
- `podinfo-application.yaml`
- `podinfo/values.yaml`

### `platform/`
Contains the observability stack:
- `kube-prometheus-stack-application.yaml`
- `kube-prometheus-stack/values.yaml`

## GitOps flow

1. Bootstrap Argo CD.
2. Apply the root Application.
3. Argo CD creates the child Applications from `apps/`.
4. podinfo deploys from the Helm chart.
5. kube-prometheus-stack deploys from the platform path.
6. Changing Git updates the cluster automatically.

## Known traps and fixes

### 1) Argo CD redirect loop on ingress
Argo CD serves UI and CLI on the same port, so behind HTTP ingress it can redirect forever unless `server.insecure=true` is set. I fixed this by setting the Helm value in `bootstrap/argocd-values.yaml`.

### 2) OCI chart access denied
I hit a `403 denied` error while pulling the podinfo chart from GHCR. I fixed this by switching to a normal Helm chart repository source for the assignment flow, which avoids the registry permission problem.

### 3) Grafana panel JSON treated as a manifest
Argo CD tried to parse raw Grafana panel JSON as Kubernetes YAML and failed with `Object 'Kind' is missing`. I fixed this by removing raw dashboard JSON from the Application source path and treating it as Grafana content, not Kubernetes manifests.

### 4) Prometheus CRD too large for client-side apply
The Prometheus CRD is large enough to hit the annotation size limit. I set `ServerSideApply=true` in the kube-prometheus-stack Application sync options to avoid the `metadata.annotations: Too long` failure.

### 5) Namespace rendering issues
Some rendered resources were missing namespaces during validation. I fixed this by explicitly setting the destination namespace and making sure namespace-related settings were applied in the Helm source.


## Design Notes

### App-of-apps vs ApplicationSet for a 50-tenant multi-client setup
For 50 tenants, I would choose **ApplicationSet**. App-of-apps is fine for small bootstraps, but ApplicationSet is better for tenant-scale generation, repeatability, and reducing manual YAML duplication. It also makes fleet-style management cleaner when tenants differ only by parameters.

### Secret management in a real stack
For real secrets, I would use **External Secrets Operator** backed by Vault or AWS Secrets Manager. That keeps secrets out of Git, supports rotation, and centralizes access control. For simple Git-only environments, Sealed Secrets is acceptable, but External Secrets is the stronger production choice.

## Validation checklist

- Argo CD UI shows all apps Healthy/Synced.
- podinfo version changes after the Git commit.
- Grafana panel displays live data from podinfo metrics.
- Prometheus target for podinfo is UP.
- Screenshots are stored in `screenshots/`.
