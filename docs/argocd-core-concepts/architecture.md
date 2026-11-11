# The Argo CD Architecture

> Three main components, each with one job: one serves you, one reads Git, one changes the cluster.

---

In [Installing Argo CD](../argocd-getting-started/installing-argocd.md) you saw about seven `argocd-` Pods. Only three of them do the core GitOps work. The others are helpers.

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
    you(["👩‍💻 you<br/>UI · argocd CLI"])
    git["📚 Git repo"]

    subgraph cluster["☸️ Kubernetes cluster"]
        direction TB
        subgraph argo["🐙 argocd namespace"]
            direction TB
            server["🖥️ argocd-server<br/>UI + API"]
            repo["📥 repo-server<br/>reads Git, renders YAML"]
            ctrl["🔁 application-controller<br/>compare · sync"]
        end
        api["🚪 kube-apiserver"]
        apps["🚀 your app<br/>Deployment, Service, …"]
    end

    you --> server
    server -->|"read / write Applications"| api
    ctrl -->|"what should run?"| repo
    repo -->|clone| git
    ctrl ==>|"watch live state · apply changes"| api
    api --> apps

    classDef ctrl fill:#0F172A,stroke:#2563EB,stroke-width:2px,color:#FFFFFF
    classDef eng fill:#D97706,stroke:#92400E,stroke-width:3px,color:#FFFFFF
    classDef tgt fill:#0F172A,stroke:#16A34A,stroke-width:2px,color:#FFFFFF
    class you,git ctrl
    class ctrl eng
    class server,repo,api,apps tgt
    linkStyle 4 stroke:#D97706,stroke-width:3px
    style cluster fill:#1E293B,stroke:#334155,color:#F1F5F9,fillStyle:solid
    style argo fill:#000000,stroke:#94A3B8,color:#FFFFFF,fillStyle:solid
```

### The three core components

| Component | Its one job | Think of it as |
|-----------|-------------|----------------|
| **argocd-server** | Serves the web UI and the API that the `argocd` CLI talks to. | the front desk |
| **argocd-repo-server** | Clones Git repos and turns them into plain Kubernetes YAML (rendering Helm or Kustomize if needed). | the reader |
| **argocd-application-controller** | Compares that YAML with what's running, reports the difference, and applies changes when it syncs. | the worker |

The **application-controller** is where GitOps actually happens: it runs the reconcile loop from [The GitOps Principles](../gitops-intro/gitops-principles.md). The other two only *show* (server) and *read* (repo-server).

### The helpers

| Component | What it does |
|-----------|--------------|
| **argocd-redis** | a cache, so Git and the cluster aren't asked the same thing over and over |
| **argocd-dex-server** | SSO login (GitHub, Google, …). Unused while we log in as `admin`. |
| **argocd-applicationset-controller** | generates many Applications from one template |
| **argocd-notifications-controller** | sends Slack/email messages on sync events |

If a helper is down, Argo CD mostly keeps working. If one of the three core components is down, something visible breaks:

| If this is down… | You'll notice |
|------------------|---------------|
| argocd-server | the UI and CLI stop responding, **but syncing continues** |
| repo-server | apps show `Unknown` / comparison errors: Argo CD can't read Git |
| application-controller | nothing syncs and drift isn't detected, even though the UI loads fine |

> When something looks wrong, ask *which job* failed: showing, reading or syncing. That tells you which Pod's logs to read, e.g. `kubectl -n argocd logs deploy/argocd-repo-server`.

### Where Argo CD keeps its state

Argo CD has no database. Everything it needs is stored as ordinary Kubernetes objects in the `argocd` namespace:

- **Applications** (a CRD): what to deploy, from where, to where. Covered on the next page.
- **ConfigMaps and Secrets**: settings, repo credentials, and the admin password hash you changed in [Accessing Argo CD](../argocd-getting-started/accessing-argocd.md).

That's why it can be installed with a Helm chart, and why backing up Argo CD mostly means backing up the `argocd` namespace.

---

[← Accessing Argo CD](../argocd-getting-started/accessing-argocd.md) · [↑ Contents](../../README.md) · [The Application CRD →](application-crd.md)
