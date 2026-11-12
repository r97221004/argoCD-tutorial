# The GitOps Workflow

> Flip the direction: instead of CI pushing into the cluster, an agent *inside* the cluster pulls from Git and keeps the cluster matching it.

---

GitOps keeps Git as the place where the desired state lives, but changes **who applies it**. The pipeline no longer touches the cluster. Instead, a controller running *in* the cluster, here Argo CD, watches a Git repo and continuously reconciles the cluster toward it. It's the same control-loop idea Kubernetes uses for Deployments, applied one level up.

```mermaid
---
config:
  look: handDrawn
  theme: default
  themeVariables:
    fontFamily: '"Comic Sans MS", "Comic Sans", "Segoe Print", "Bradley Hand", cursive'
    clusterBkg: '#FAFAFA'
    clusterBorder: '#94A3B8'
    lineColor: '#94A3B8'
    edgeLabelBackground: '#475569'
  themeCSS: |
    .edgeLabel, .edgeLabel p, .edgeLabel span { color: #FFFFFF !important; }
---
flowchart LR
    dev(["👩‍💻 developer"])
    app["📦 app repo<br/>source code"]

    subgraph ci["⚙️ CI · build only"]
        build["🔨 build, test,<br/>push image"]
    end

    reg[("🗃️ image registry")]
    cfg["📚 config repo<br/>Kubernetes YAML<br/>= desired state"]

    subgraph cluster["☸️ Kubernetes cluster"]
        direction TB
        argo["🐙 Argo CD<br/>compare · sync · heal"]
        live["🚀 live resources"]
        argo -->|apply| live
        live -. "observe" .-> argo
    end

    dev -->|git push| app
    app --> build
    build --> reg
    build -->|"bump image tag<br/>(commit / PR)"| cfg
    argo ==>|"PULL · watches"| cfg
    live -. pulls image .-> reg

    classDef ctrl fill:#0F172A,stroke:#2563EB,stroke-width:2px,color:#FFFFFF
    classDef eng fill:#D97706,stroke:#92400E,stroke-width:3px,color:#FFFFFF
    classDef tgt fill:#0F172A,stroke:#16A34A,stroke-width:2px,color:#FFFFFF
    class dev,app,cfg ctrl
    class argo eng
    class build,reg,live tgt
    linkStyle 6 stroke:#D97706,stroke-width:3px
    style ci fill:#000000,stroke:#94A3B8,color:#FFFFFF,fillStyle:solid
    style cluster fill:#1E293B,stroke:#334155,color:#F1F5F9,fillStyle:solid
```

> **Why two repos?** The **app repo** holds code and triggers CI. The **config repo** holds the Kubernetes YAML that Argo CD watches. Keeping them apart means a code change alone never deploys anything. Only a change to the config repo does.

### One change, end to end

1. **Open a PR.** A developer pushes a branch and opens a PR in the **app repo**. CI runs lint and unit tests, and the PR is merged once they pass.
2. **CI builds, and stops there.** On merge to `main`, CI builds the image and pushes it, such as `myapp:1.4.2`, to the registry. It never touches the cluster.
3. **Update Git.** CI commits the new image tag to the **config repo**, or opens a PR for someone to review. CI now needs write access to Git, not to the cluster.
4. **Argo CD notices.** It checks the repo every 3 minutes by default and sees that Git and the cluster differ: the app is **`OutOfSync`**.
5. **Argo CD syncs.** With automated sync it applies the change on its own; otherwise it waits for you to press Sync. Either way it applies from *inside* the cluster and waits until it's **`Healthy`**.
6. **Argo CD keeps watching.** If someone changes the cluster with `kubectl` later, Argo CD marks the app `OutOfSync` and, with self-heal turned on, changes it back.

> **Other ways to update the config repo.** You can also edit the YAML by hand, run a workflow inside the config repo, or let a tool like Argo CD Image Updater do it. Whatever the method, the change has to land in Git.

### Push vs. pull

| | Push (CI deploys) | Pull (GitOps) |
|---|---|---|
| Who applies changes | CI, from outside the cluster | Argo CD, from inside |
| Cluster credentials | stored in CI | never leave the cluster |
| After the deploy | the job exits | keeps comparing and reports health |
| Drift | goes unnoticed | detected, and can be reverted |
| Audit trail | pipeline logs, plus whatever `kubectl` did | `git log`: every change has an author and a reason |
| Rollback | re-run an old pipeline | `git revert` |

> The key idea: **every change to the cluster is a Git commit.** Deploying, rolling back and auditing all become Git operations you already know: PR, merge, revert, log.

---

[← The Push Model and Its Drawbacks](push-model.md) · [↑ Contents](../../README.md) · [The GitOps Principles →](gitops-principles.md)
