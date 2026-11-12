# Sync and Health Checks

> Every Application has two separate statuses: does the cluster *match Git*, and is the app *working*?

---

In the [first lab](applications-vs-manifests.md#lab-deploy-your-first-application) you saw two lines: `Sync Status` and `Health Status`. They answer different questions, and they change independently.

### Sync status: does the cluster match Git?

| Status | Meaning |
|--------|---------|
| **Synced** | The live resources match the manifests in Git. |
| **OutOfSync** | Something differs: a new commit in Git, or a change made directly in the cluster. |
| **Unknown** | Argo CD couldn't compare, usually because it can't read the repo or the YAML is invalid. |

### Health status: is the app working?

| Status | Meaning | Example |
|--------|---------|---------|
| **Healthy** | Running as expected. | Deployment has all replicas ready |
| **Progressing** | Not ready yet, but on its way. | Pods are starting during a rollout |
| **Degraded** | Failed. | a rollout that never became ready before its deadline (10 minutes by default) |
| **Missing** | In Git, but not in the cluster. | before the first sync |
| **Suspended** | Paused on purpose. | a suspended CronJob |

Argo CD has built-in health checks for common kinds. For example, a Deployment is Healthy when its updated replicas are all available.

### Why two statuses?

Because each can be fine while the other isn't:

| Sync | Health | What's going on |
|------|--------|-----------------|
| Synced | Healthy | ✅ Git is deployed and works. |
| Synced | Progressing → Degraded | Git was applied correctly, but what's *in* Git is broken, e.g. an image tag that doesn't exist: Progressing at first, Degraded once the rollout times out. **Fix Git.** |
| OutOfSync | Healthy | The app works, but doesn't match Git: a new commit isn't synced yet, or someone changed the cluster by hand. |
| OutOfSync | Missing | Nothing deployed yet, like the first lab before you pressed Sync. |

> "Synced" doesn't mean "working", and "Healthy" doesn't mean "matches Git". Check both.

### Refresh vs. sync

Two buttons in the UI that are easy to mix up:

| | Refresh | Sync |
|---|---|---|
| What it does | Re-reads Git and **re-compares**. Changes nothing. | **Applies** Git to the cluster. |
| When it happens on its own | every 3 minutes by default | only with automated sync |
| CLI | `argocd app get <app> --refresh` | `argocd app sync <app>` |

Changes made directly in the cluster show up right away, because Argo CD watches the cluster continuously. New commits in Git are only noticed on the next refresh, so after a `git push`, press **Refresh** if you don't want to wait.

### Sync policy: who presses Sync?

| Setting | Effect |
|---------|--------|
| *(none)* | **Manual.** Argo CD reports OutOfSync and waits for you. |
| `automated: {}` | Syncs automatically when **Git** changes. |
| `automated.selfHeal: true` | Also reverts changes made directly in the **cluster**. |
| `automated.prune: true` | Also deletes resources that were **removed from Git**. |

By default, automated sync **doesn't** self-heal and **doesn't** prune. Both are opt-in, so Argo CD never deletes or overwrites something unless you asked for it. You'll try them in the lab below.

### Lab: The Sync Process

> Change the app through Git, break it through `kubectl`, then let Argo CD fix it on its own.

<details>
<summary><b>Show the lab steps</b></summary>

**Goal:** see a Git change go out with a manual sync, catch drift, then turn on automated sync and self-heal.

**Before you start:** `guestbook` from [Lab: Deploy Your First Application](applications-vs-manifests.md#lab-deploy-your-first-application) is `Synced` and `Healthy`. You have both clones on the VM: your fork of `argocd-apps` and this tutorial.

> 💡 Keep k9s open in a second terminal (`k9s -n guestbook`, then `:pods`) to watch Pods appear and disappear as you go.

#### 1. Change Git, sync by hand

Scale guestbook from 1 to 2 replicas **in Git**. In your fork, edit `guestbook/guestbook-ui-deployment.yaml`:

```yaml
spec:
  replicas: 2
```

Commit and push:

```bash
cd argocd-apps
git commit -am "scale guestbook-ui to 2 replicas"
git push
```

Argo CD checks Git every 3 minutes. Refresh now instead of waiting:

```bash
argocd app get guestbook --refresh   # Sync Status: OutOfSync
argocd app diff guestbook            # shows replicas: 1 → 2
```

The cluster still has 1 Pod: with a manual policy, nothing happens until you sync.

```bash
argocd app sync guestbook
kubectl get pods -n guestbook        # now 2 Pods
```

#### 2. Create drift, fix it by hand

Now change the cluster directly, the way [the push model](../gitops-intro/push-model.md#configuration-drift) can't notice:

```bash
kubectl scale deploy guestbook-ui -n guestbook --replicas=5
```

Argo CD notices right away, with no refresh needed:

```bash
argocd app get guestbook             # Sync Status: OutOfSync
argocd app diff guestbook            # live: 5, Git: 2
```

This is the difference from the push model: the drift is **visible**. Put it back by syncing:

```bash
argocd app sync guestbook
kubectl get deploy guestbook-ui -n guestbook   # READY 2/2
```

#### 3. Turn on automated sync and self-heal

Pressing Sync every time doesn't scale. In the tutorial clone, add a `syncPolicy.automated` block to `manifests/argocd-core-concepts/application.yaml`:

```yaml
  syncPolicy:
    automated:
      selfHeal: true # revert changes made directly in the cluster
    syncOptions:
      - CreateNamespace=true
```

Apply the updated Application:

```bash
cd ../argoCD-tutorial
kubectl apply -f manifests/argocd-core-concepts/application.yaml
```

Create the same drift again:

```bash
kubectl scale deploy guestbook-ui -n guestbook --replicas=5
kubectl get deploy guestbook-ui -n guestbook -w   # watch it go 5 → back to 2
```

Within a few seconds Argo CD scales it back to 2, with no command from you. This is the [continuously reconciled](../gitops-intro/gitops-principles.md) principle in action. Press `Ctrl-C` to stop watching.

Git changes are now deployed automatically too. In your fork, set `replicas: 1` again, then commit, push and refresh. The cluster follows on its own:

```bash
cd ../argocd-apps
git commit -am "scale guestbook-ui back to 1 replica"
git push
argocd app get guestbook --refresh
kubectl get pods -n guestbook        # back to 1 Pod
```

> **About `prune`.** We left it off. If you delete `guestbook-ui-svc.yaml` from Git now, Argo CD shows the Service as OutOfSync but **won't delete it**. Add `prune: true` under `automated` when you want deletions in Git to delete things in the cluster too.

#### What you saw

| Change came from | Manual policy | Automated + selfHeal |
|------------------|---------------|----------------------|
| **Git** (commit + push) | OutOfSync until you sync | deployed automatically |
| **Cluster** (`kubectl scale`) | OutOfSync until you sync | reverted within seconds |

#### Clean up

Delete the Application. `argocd app delete` also deletes everything it deployed:

```bash
argocd app delete guestbook --yes
kubectl get all -n guestbook                   # resources are gone; the namespace may remain
kubectl delete ns guestbook --ignore-not-found
```

The namespace may stay behind because Argo CD created it through `CreateNamespace=true` and doesn't track it as part of the app. Deleting it by hand is safe.

</details>

---

[← Applications vs. Kubernetes Manifests](applications-vs-manifests.md) · [↑ Contents](../../README.md)
