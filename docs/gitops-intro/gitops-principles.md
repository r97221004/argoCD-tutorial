# The GitOps Principles

> Four rules that turn "we keep YAML in Git" into GitOps, and how Argo CD implements each one.

---

"GitOps" gets used loosely. Storing manifests in Git and running `kubectl apply` from CI is still the push model from the [previous page](push-model.md). The [OpenGitOps](https://opengitops.dev/) project (part of the CNCF) defines GitOps as a set of **four principles**. A system is GitOps only when it meets all four.

### 1. Declarative

> _A system managed by GitOps must have its desired state expressed declaratively._

You describe **what** the system should look like, not the steps to get there. Kubernetes manifests already work this way: `replicas: 3`, not "start three more Pods".

**In Argo CD**: it deploys plain YAML, Kustomize or Helm. All three render to declarative manifests. A shell script that runs `kubectl` commands can't be managed this way.

### 2. Versioned and Immutable

> _Desired state is stored in a way that enforces immutability, versioning and retains a complete version history._

The desired state lives in a store where every change is a new, permanent version. In practice that's Git: each commit is immutable, has an author and a message, and can be reverted.

**In Argo CD**: an Application points at a repo, a path and a **revision** (branch, tag or commit SHA). Every sync records which commit it deployed. Rolling back means `git revert` or pointing at an older revision.

### 3. Pulled Automatically

> _Software agents automatically pull the desired state declarations from the source._

An agent fetches the desired state itself, instead of an outside system pushing it in. This is what removes cluster credentials from CI and stops the API server from needing to accept connections from the pipeline.

**In Argo CD**: Argo CD checks the repo every 3 minutes by default, or right away when a webhook arrives. The **repo-server** fetches the repo and renders the manifests.

### 4. Continuously Reconciled

> _Software agents continuously observe actual system state and attempt to apply the desired state._

The agent doesn't stop after one deploy. It keeps comparing live state with desired state and acts on any difference, whether the change came from Git or from someone running `kubectl`.

**In Argo CD**: the **application-controller** runs this loop. It marks an app `Synced` or `OutOfSync`, and with **automated sync** and **self-heal** turned on, it reverts drift such as the `kubectl scale` from the [drift demo](push-model.md#try-it--create-drift-by-hand).

### Putting them together

| Principle | Fixes this push-model problem | Argo CD piece |
|-----------|------------------------------|---------------|
| Declarative | imperative scripts nobody can diff | YAML / Kustomize / Helm sources |
| Versioned & Immutable | "what's running now?" has no answer | Git revision on every Application |
| Pulled Automatically | CI holds cluster-admin credentials | repo-server pulling from Git |
| Continuously Reconciled | drift goes unnoticed | application-controller sync loop |

> Principles 3 and 4 are what make it GitOps rather than just "YAML in Git". They are the reason Argo CD runs *inside* the cluster, which is what you installed in [Environment Setup](../environment-setup/install-argocd.md).

---

[← The GitOps Workflow](gitops-workflow.md) · [↑ Contents](../../README.md) · [Getting Started with Argo CD →](../argocd-getting-started/overview.md)
