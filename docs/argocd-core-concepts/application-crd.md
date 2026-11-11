# The Application CRD

> An Application is how you tell Argo CD what to deploy: *this* folder in Git, to *that* cluster and namespace.

---

Installing Argo CD added a new resource type to Kubernetes: **`Application`** (a CRD, Custom Resource Definition). You write it in YAML and `kubectl apply` it like a Deployment. The difference is who reads it: not Kubernetes, but the Argo CD application-controller.

Here's the one you'll deploy in the [lab](first-application.md), [`manifests/argocd-core-concepts/application.yaml`](../../manifests/argocd-core-concepts/application.yaml):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_GITHUB_USER>/<GITOPS_REPO>.git
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
| `metadata.namespace: argocd` | Applications always live in **Argo CD's own namespace**, not where your app runs. |
| `spec.project` | The AppProject this app belongs to. `default` allows everything, which is fine for now. |
| **Where from: `spec.source`** | |
| `repoURL` | The Git repo to read. Here it's a separate config repo, not this tutorial (see [Two repos](../gitops-intro/gitops-workflow.md)). |
| `targetRevision` | Which branch, tag or commit to follow. `HEAD` means "always the latest commit on the default branch". |
| `path` | The folder inside the repo that holds the manifests. |
| **Where to: `spec.destination`** | |
| `server` | Which cluster. `https://kubernetes.default.svc` means "the cluster Argo CD itself runs in". |
| `namespace` | Which namespace your app's resources go into. |
| **How: `spec.syncPolicy`** | |
| `syncOptions: CreateNamespace=true` | Create the destination namespace if it doesn't exist. |
| *(no `automated:` yet)* | Without it, Argo CD only *reports* differences and waits for you to press Sync. You'll turn it on in [The Sync Process](sync-process.md). |

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
