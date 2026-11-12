# The Application CRD

> An Application is how you tell Argo CD what to deploy: *this* folder in Git, to *that* cluster and namespace.

---

Installing Argo CD added a new resource type to Kubernetes: **`Application`** (a CRD, Custom Resource Definition). You write it in YAML and `kubectl apply` it like a Deployment. The difference is who reads it: not Kubernetes, but the Argo CD application-controller.

You can see the new type for yourself:

```bash
kubectl get crd applications.argoproj.io   # the CRD Argo CD installed
kubectl explain application.spec.source    # its fields, documented like any built-in resource
```

Here's the one you'll deploy in the [lab on the next page](applications-vs-manifests.md#lab-deploy-your-first-application), [`manifests/argocd-core-concepts/application.yaml`](../../manifests/argocd-core-concepts/application.yaml):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_GITHUB_USER>/argocd-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

### Field by field

It answers three questions: **where from**, **where to**, and **how**.

| Field | Meaning |
|-------|---------|
| `metadata.namespace: argocd` | By default, Applications must live in **Argo CD's own namespace**, not where your app runs. |
| `spec.project` | The AppProject this app belongs to. A project limits which repos an app may use, which clusters and namespaces it may deploy to, and which resource kinds it may create. `default` allows everything, which is fine for now. |
| **Where from: `spec.source`** | |
| `repoURL` | The Git repo to read. Here it's a separate config repo, not this tutorial (see [Two repos](../gitops-intro/gitops-workflow.md)). |
| `targetRevision` | Which branch, tag or commit to follow. `HEAD` means "always the latest commit on the default branch". |
| `path` | The folder inside the repo that holds the manifests: plain YAML, a Kustomize overlay or a Helm chart. The repo-server detects which and renders it to YAML. (To use a chart from a Helm repository instead of Git, replace `path` with `chart:` and set `targetRevision` to the chart version.) |
| **Where to: `spec.destination`** | |
| `server` | Which cluster. `https://kubernetes.default.svc` means "the cluster Argo CD itself runs in". |
| `namespace` | Which namespace your app's resources go into. |
| **How: `spec.syncPolicy`** *(optional)* | |
| `syncOptions: CreateNamespace=true` | Create the destination namespace if it doesn't exist. |
| *(no `automated:` yet)* | Without it, Argo CD only *reports* differences and waits for you to press Sync. `automated`, plus its `prune` and `selfHeal` switches, are covered in [Sync and Health Checks](sync-and-health.md#sync-policy-who-presses-sync). |

### Three ways to create one

| How | Example | GitOps-friendly? |
|-----|---------|------------------|
| **YAML + `kubectl apply`** | `kubectl apply -f application.yaml` | ✅ The Application itself can live in Git |
| `argocd` CLI | `argocd app create guestbook --repo … --path … --dest-namespace …` | ⚠️ Only exists in the cluster |
| Web UI | **+ NEW APP** button | ⚠️ Only exists in the cluster |

All three produce the same `Application` object. This guide uses YAML, so the Application is versioned like everything else. The UI and CLI are great for exploring.

```bash
kubectl get applications -n argocd   # list them with kubectl…
argocd app list                      # …or with the CLI
```

---

[← The Argo CD Architecture](architecture.md) · [↑ Contents](../../README.md) · [Applications vs. Kubernetes Manifests →](applications-vs-manifests.md)
