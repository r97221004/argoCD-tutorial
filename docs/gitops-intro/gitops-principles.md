# The GitOps Principles

> Four rules that turn "we keep YAML in Git" into GitOps, and how Argo CD implements each one.

---

Storing manifests in Git and running `kubectl apply` from CI is *not* GitOps. That's still the push model from the [first page](push-model.md). The [OpenGitOps](https://opengitops.dev/) project (part of the CNCF) defines GitOps with **four principles**:

| # | Principle | In plain words | In Argo CD |
|---|-----------|----------------|------------|
| 1 | **Declarative** | Describe *what* you want (`replicas: 3`), not the steps to get there. | Apps are plain YAML, Kustomize or Helm |
| 2 | **Versioned and Immutable** | Git keeps every version, and a commit never changes once made, so you can review, revert, or return to any exact earlier state. | Each Application points at a Git repo and revision: a branch to follow, or a tag or commit to pin an exact version |
| 3 | **Pulled Automatically** | An agent fetches the desired state itself. Nobody pushes into the cluster. | Checks the repo every 3 minutes (or on a webhook) |
| 4 | **Continuously Reconciled** | The agent keeps comparing Git with the cluster and works to make them match. | Always compares and shows `Synced` / `OutOfSync`; with automated sync and self-heal it also reverts drift like the one in the [demo](push-model.md#try-it--create-drift-by-hand) |

Principles 1 and 2 you may already follow with "YAML in Git". **Principles 3 and 4 are what make it GitOps.** They're the reason Argo CD runs *inside* the cluster.

### Two things to know before you start

**1. Don't put Secrets in Git.** "Everything in Git" has one big exception. A Kubernetes Secret is only base64-*encoded*, not encrypted, so committing one publishes the password to anyone who can read the repo. GitOps setups use a tool that keeps only an *encrypted* version, or a *reference* to a secret store, in Git. Sealed Secrets and External Secrets are common ones. For now, just remember: **never commit a plain Secret.**

**2. Manual changes don't stick.** With self-heal turned on, Argo CD reverts any change that isn't in Git, including your 2 a.m. `kubectl edit` hotfix. That's the whole point, but it surprises everyone the first time. In GitOps, a fix isn't done until it's committed.

---

[← The GitOps Workflow](gitops-workflow.md) · [↑ Contents](../../README.md) · [Installing Argo CD →](../argocd-getting-started/installing-argocd.md)
