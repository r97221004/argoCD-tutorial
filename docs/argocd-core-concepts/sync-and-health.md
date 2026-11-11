# Sync and Health Checks

> Every Application has two separate statuses: does the cluster *match Git*, and is the app *working*?

---

In the [first lab](first-application.md) you saw two lines: `Sync Status` and `Health Status`. They answer different questions, and they change independently.

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

By default, automated sync **doesn't** self-heal and **doesn't** prune. Both are opt-in, so Argo CD never deletes or overwrites something unless you asked for it. You'll try each of these in the next lab.

---

[← Lab: Deploying Your First Application](first-application.md) · [↑ Contents](../../README.md) · [Lab: The Sync Process →](sync-process.md)
