# Lab: Deploying Your First Application

> Fork the config repo, point an Application at its `guestbook/` folder, and let Argo CD deploy it.

---

**Goal:** deploy **guestbook**, Argo CD's classic example app (a `guestbook-ui` Deployment and Service), without ever running `kubectl apply` on the app itself.

**Before you start:** you're logged in to Argo CD (`argocd app list` works), see [Accessing Argo CD](../argocd-getting-started/accessing-argocd.md). Run the commands on the VM.

### 1. Get the two repos

This lab uses the [two-repo layout](../gitops-intro/gitops-workflow.md) from the GitOps chapter:

| Repo | Holds | You need |
|------|-------|----------|
| **`<GITOPS_REPO>`** (config repo) | the guestbook manifests in `guestbook/` | **your own fork**: in the [next lab](sync-process.md) you'll push changes to it |
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

### 2. Point the Application at your fork

In the tutorial clone, edit [`manifests/argocd-core-concepts/application.yaml`](../../manifests/argocd-core-concepts/application.yaml) and replace `<YOUR_GITHUB_USER>` with your GitHub account:

```yaml
    repoURL: https://github.com/<YOUR_GITHUB_USER>/<GITOPS_REPO>.git
```

### 3. Create the Application

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

### 4. Sync

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

### 5. Look at what Argo CD did

```bash
kubectl get deploy,svc,pods -n guestbook
```

The namespace, the `guestbook-ui` Deployment and Service, and its Pod all exist. You never applied them: Argo CD did, from your fork. In the UI, the app's tree view shows the same thing: **Application → Deployment → ReplicaSet → Pod**, plus the Service.

Want to see the guestbook page itself? Port-forward the Service and open `http://localhost:8081` (tunnel it with `ssh -L` if the VM is remote, as in [Accessing Argo CD](../argocd-getting-started/accessing-argocd.md)):

```bash
kubectl -n guestbook port-forward svc/guestbook-ui 8081:80
```

### Checkpoint

| You did | Which means |
|---------|-------------|
| `kubectl apply` the **Application** | you told Argo CD *where* to look |
| `argocd app sync` | you told it to make the cluster match Git, **once** |
| Never applied the guestbook YAML | Git is now the only source of the app |

Leave the app running: the [next lab](sync-process.md) changes it.

---

[← Applications vs. Kubernetes Manifests](applications-vs-manifests.md) · [↑ Contents](../../README.md) · [Sync and Health Checks →](sync-and-health.md)
