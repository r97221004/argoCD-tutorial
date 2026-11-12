# Deploying Public Helm Charts

> A public chart lives in a Helm repository, not in Git. The Application names the repo, the chart and an exact version.

---

In [Helm Charts in Argo CD](helm-charts.md) the chart sat in your Git repo. Most charts you'll use are published by someone else, in a **Helm repository**: a web server with an `index.yaml` listing every chart version. Argo CD can deploy straight from one, without copying the chart into Git.

### Git source vs. Helm repository source

| Field | Chart in Git | Chart in a Helm repository |
|-------|--------------|----------------------------|
| `repoURL` | the Git repo (`https://github.com/…/argocd-apps.git`) | the Helm repo (`https://kubernetes-sigs.github.io/headlamp/`) |
| which chart | `path`: the folder with `Chart.yaml` | `chart`: the chart's **name** in the repo |
| `targetRevision` | a branch, tag or commit (`HEAD`) | a **chart version** (`0.45.0`) |

Everything else in the Application (destination, sync policy) is the same. The chart in this lab is **Headlamp**, a web UI for Kubernetes from the Kubernetes SIG UI project and the recommended replacement for the retired Kubernetes Dashboard:

```yaml
  source:
    repoURL: https://kubernetes-sigs.github.io/headlamp/   # a Helm repository
    chart: headlamp                                        # instead of path
    targetRevision: 0.45.0                                 # the chart version, pinned
```

> **Chart version ≠ app version.** `targetRevision` is the version of the **chart** (its packaging). The software inside has its own `appVersion`. They often differ; for Headlamp they happen to match.

### The "latest" trap: pin the chart version

For a Helm repository, `targetRevision` also accepts ranges such as `*` (any version) or `0.*`. Argo CD then picks the newest matching version **every time it refreshes**. That sounds convenient, but:

- A chart release you never reviewed reaches your cluster, with no commit in Git to show it happened. Git no longer says what's running, which breaks the [*versioned and immutable*](../gitops-intro/gitops-principles.md) principle.
- You can't roll back by reverting a commit, because there is no commit.
- The same Application deploys different things on different days, so staging and production can quietly diverge.

**Pin an exact version.** Upgrading then becomes a one-line change in Git that you review, sync and can revert, which is what the lab does.

### Lab: Deploy Headlamp from Its Helm Repository

> Deploy a pinned Headlamp chart, log in to it, then upgrade it by changing one line.

<details>
<summary><b>Show the lab steps</b></summary>

**Goal:** deploy a chart straight from a Helm repository, then upgrade it the GitOps way: change the version, review the diff, sync.

**Before you start:** you're logged in to Argo CD and have the tutorial cloned on the VM. Nothing needs forking: the chart comes from Headlamp's public Helm repo.

#### 1. Look at the chart before deploying it

```bash
helm show chart headlamp --repo https://kubernetes-sigs.github.io/headlamp/ --version 0.45.0
# appVersion: 0.45.0   ← the Headlamp release inside it
# name: headlamp
# version: 0.45.0      ← the chart version: what targetRevision pins
```

#### 2. Create the Application

[`manifests/argocd-helm/headlamp.yaml`](../../manifests/argocd-helm/headlamp.yaml) needs no edits:

```bash
cd ~/argoCD-tutorial   # wherever you cloned the tutorial
kubectl apply -f manifests/argocd-helm/headlamp.yaml
argocd app sync headlamp
argocd app wait headlamp --health
kubectl get deploy,svc -n headlamp   # headlamp Deployment and Service
```

#### 3. Log in to Headlamp

Port-forward the Service and open `http://localhost:8082` (tunnel it with `ssh -L 8082:localhost:8082 …` if the VM is remote, as in [Accessing Argo CD](../argocd-getting-started/accessing-argocd.md)):

```bash
kubectl -n headlamp port-forward svc/headlamp 8082:80
```

Headlamp asks for a token. In a second terminal, create one for the ServiceAccount the chart installed:

```bash
kubectl create token headlamp -n headlamp
```

Paste it, and you can browse the whole cluster, including the `helm-guestbook` app from the previous lab.

> ⚠️ By default this chart binds its ServiceAccount to **`cluster-admin`**, so anyone with that token controls the cluster. Fine for a lab VM; in production, set `clusterRoleBinding.create: false` and grant narrower rights. The next chapter shows how to set chart values like that one.

#### 4. Upgrade the chart

Headlamp `0.46.0` is out. In `manifests/argocd-helm/headlamp.yaml`, change one line:

```yaml
    targetRevision: 0.46.0
```

Apply the Application and look at what would change before syncing:

```bash
kubectl apply -f manifests/argocd-helm/headlamp.yaml
argocd app get headlamp --refresh   # Sync Status: OutOfSync
argocd app diff headlamp            # image headlamp:v0.45.0 → v0.46.0, chart labels 0.45.0 → 0.46.0
argocd app sync headlamp
argocd app wait headlamp --health
```

The upgrade happened because you changed a version and reviewed the diff, not because a new chart was published. To roll back, set `0.45.0` again and sync.

> In this guide the Application file is applied with `kubectl`. In a real project it lives in Git ([App of Apps](../argocd-core-concepts/application-crd.md#three-ways-to-create-one)), so this upgrade is a reviewed commit like any other change.

#### Checkpoint

| You did | Which means |
|---------|-------------|
| `repoURL` + `chart` + `targetRevision: 0.45.0` | deployed from a Helm repository, at an exact chart version |
| Changed `targetRevision` to `0.46.0` | upgrades are deliberate, reviewable one-line changes |
| `argocd app diff` before syncing | you saw the new image before it reached the cluster |

Leave `headlamp` running if you like: it's handy for watching the next lab. [Overriding Chart Values](chart-values.md) cleans up both apps at the end.

</details>

---

[← Helm Charts in Argo CD](helm-charts.md) · [↑ Contents](../../README.md) · [Overriding Chart Values →](chart-values.md)
