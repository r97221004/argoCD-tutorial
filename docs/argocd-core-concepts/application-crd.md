# The Application CRD

> An Application is how you tell Argo CD what to deploy: *this* folder in Git, to *that* cluster and namespace.

---

Installing Argo CD added a new resource type to Kubernetes: **`Application`** (a CRD, Custom Resource Definition). You write it in YAML and `kubectl apply` it like a Deployment. The difference is who reads it: not Kubernetes, but the Argo CD application-controller.

You can see the new type for yourself:

```bash
kubectl get crd applications.argoproj.io   # the CRD Argo CD installed
kubectl explain application.spec.source    # its fields, documented like any built-in resource
```

Here's the one you'll deploy in the [lab below](#lab-deploy-your-first-application), [`manifests/argocd-core-concepts/application.yaml`](../../manifests/argocd-core-concepts/application.yaml):

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

> **Deleting an Application.** `argocd app delete` (and the UI) also deletes the resources the Application deployed. A plain `kubectl delete application` does **not**: it removes only the pointer and leaves the app running, unless the Application has the finalizer `resources-finalizer.argocd.argoproj.io` in `metadata.finalizers`.

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

### Lab: Deploy Your First Application

> Fork the config repo, point an Application at its `guestbook/` folder, and let Argo CD deploy it.

<details>
<summary><b>Show the lab steps</b></summary>

**Goal:** deploy **guestbook**, Argo CD's classic example app (a `guestbook-ui` Deployment and Service), without ever running `kubectl apply` on the app itself.

**Before you start:** you're logged in to Argo CD (`argocd app list` works), see [Accessing Argo CD](../argocd-getting-started/accessing-argocd.md). Run the commands on the VM.

#### 1. Get the two repos

This lab uses the [two-repo layout](../gitops-intro/gitops-workflow.md) from the GitOps chapter:

| Repo | Holds | You need |
|------|-------|----------|
| **`<GITOPS_REPO>`** (config repo) | the guestbook manifests in `guestbook/` | **your own fork**: in the [sync lab](sync-and-health.md#lab-the-sync-process) you'll push changes to it |
| **this tutorial** | the Application in `manifests/argocd-core-concepts/application.yaml` | a clone |

1. Fork **`https://github.com/r97221004/<GITOPS_REPO>`** on GitHub. Keep the fork **public** so Argo CD can read it without credentials.
2. Clone both onto the VM:

   ```bash
   git clone https://github.com/<YOUR_GITHUB_USER>/<GITOPS_REPO>.git
   git clone https://github.com/r97221004/argoCD-tutorial.git
   ```

The app Argo CD will deploy is in your fork:

```bash
ls <GITOPS_REPO>/guestbook/   # guestbook-ui-deployment.yaml  guestbook-ui-svc.yaml
```

#### 2. Point the Application at your fork

In the tutorial clone, edit [`manifests/argocd-core-concepts/application.yaml`](../../manifests/argocd-core-concepts/application.yaml) and replace `<YOUR_GITHUB_USER>` with your GitHub account:

```yaml
    repoURL: https://github.com/<YOUR_GITHUB_USER>/<GITOPS_REPO>.git
```

#### 3. Create the Application

```bash
cd argoCD-tutorial
kubectl apply -f manifests/argocd-core-concepts/application.yaml
```

This is the only `kubectl apply` in the lab, and it creates the *pointer*, not the app. Check what Argo CD thinks:

```bash
argocd app get guestbook
```

```
Sync Status:        OutOfSync from HEAD (…)
Health Status:      Missing
```

- **OutOfSync**: Git has a Deployment and a Service, but the cluster doesn't.
- **Missing**: the resources don't exist yet.

Nothing was deployed, because the Application has no automated sync. Argo CD only *reports* the difference. Confirm it:

```bash
kubectl get ns guestbook   # NotFound
```

#### 4. Sync

Tell Argo CD to make the cluster match Git:

```bash
argocd app sync guestbook
```

Or in the UI: open **guestbook** → **SYNC** → **SYNCHRONIZE**.

Then wait until it's healthy:

```bash
argocd app wait guestbook --health
argocd app get guestbook
```

```
Sync Status:        Synced to HEAD (…)
Health Status:      Healthy
```

#### 5. Look at what Argo CD did

```bash
kubectl get deploy,svc,pods -n guestbook
```

The namespace, the `guestbook-ui` Deployment and Service, and its Pod all exist. You never applied them: Argo CD did, from your fork. In the UI, the app's tree view shows the same thing: **Application → Deployment → ReplicaSet → Pod**, plus the Service.

Want to see the guestbook page itself? Port-forward the Service and open `http://localhost:8081` (tunnel it with `ssh -L` if the VM is remote, as in [Accessing Argo CD](../argocd-getting-started/accessing-argocd.md)):

```bash
kubectl -n guestbook port-forward svc/guestbook-ui 8081:80
```

#### Checkpoint

| You did | Which means |
|---------|-------------|
| `kubectl apply` the **Application** | you told Argo CD *where* to look |
| `argocd app sync` | you told it to make the cluster match Git, **once** |
| Never applied the guestbook YAML | Git is now the only source of the app |

Leave the app running: the [sync lab](sync-and-health.md#lab-the-sync-process) changes it.

</details>

---

[← The Argo CD Architecture](architecture.md) · [↑ Contents](../../README.md) · [Applications vs. Kubernetes Manifests →](applications-vs-manifests.md)
