# The Push Model and Its Drawbacks

> The usual way to deploy is for CI to push changes into the cluster. It works until the cluster quietly stops matching what you think you deployed.

---

Before GitOps, the usual way to deploy to Kubernetes is a **push model**: a CI/CD pipeline (Jenkins, GitHub Actions, GitLab CI, …) builds the image, then reaches *into* the cluster and runs `kubectl apply` or `helm upgrade`.

```mermaid
---
config:
  look: handDrawn
  theme: default
  themeVariables:
    fontFamily: '"Comic Sans MS", "Comic Sans", "Segoe Print", "Bradley Hand", cursive'
    clusterBkg: '#FAFAFA'
    clusterBorder: '#94A3B8'
    lineColor: '#FFFFFF'
    edgeLabelBackground: '#475569'
  themeCSS: |
    .edgeLabel, .edgeLabel p, .edgeLabel span { color: #FFFFFF !important; }
---
flowchart LR
    dev(["👩‍💻 developer"])
    git["📚 Git repo<br/>app code + YAML"]

    subgraph ci["⚙️ CI/CD pipeline · outside the cluster"]
        direction TB
        build["🔨 build &amp; test"]
        push["📤 kubectl apply / helm upgrade<br/>🔑 holds cluster credentials"]
        build --> push
    end

    subgraph cluster["☸️ Kubernetes cluster"]
        api["🚪 kube-apiserver"]
    end

    ops(["🧑‍🔧 someone with kubectl"])

    dev -->|git push| git
    git -->|triggers| build
    push ==>|"PUSH · once per run"| api
    ops -. "kubectl edit / scale<br/>(not in Git)" .-> api

    classDef ctrl fill:#0F172A,stroke:#2563EB,stroke-width:2px,color:#FFFFFF
    classDef eng fill:#D97706,stroke:#92400E,stroke-width:3px,color:#FFFFFF
    classDef tgt fill:#0F172A,stroke:#16A34A,stroke-width:2px,color:#FFFFFF
    class dev,ops,git ctrl
    class push eng
    class build,api tgt
    linkStyle 3 stroke:#D97706,stroke-width:3px
    style ci fill:#000000,stroke:#94A3B8,color:#FFFFFF,fillStyle:solid
    style cluster fill:#1E293B,stroke:#334155,color:#F1F5F9,fillStyle:solid
```

This is the same **imperative** habit as running `kubectl apply` by hand. It's just automated: something outside the cluster issues a command, and the job is considered done once the command succeeds.

### The drawbacks

| Problem | Why it happens in a push model |
|---------|-------------------------------|
| 🔑 **Credentials live outside the cluster** | The pipeline needs a kubeconfig with write access, often cluster-admin. Every CI runner, secret store and plugin that can read it can change production. |
| 🌐 **The cluster must be reachable from CI** | The API server has to be open to the pipeline's network. That's harder with private clusters, and gets worse with every extra cluster you add. |
| 🎯 **"Deployed" means "the command succeeded"** | `kubectl apply` returning 0 doesn't mean the Pods became healthy. Once the job ends, nobody is watching. |
| 🧭 **Git is not the source of truth** | Anyone with `kubectl` can change the cluster directly. Git shows what was pushed *once*, not what's running *now*. |
| 🔁 **Nothing corrects changes** | The pipeline only runs on a new commit. If the cluster is changed in between, the change just stays. |
| ⏪ **Rollback means re-running the pipeline** | You redeploy an old build or run `kubectl rollout undo`. Both are separate actions, and neither leaves a record in Git. |

The last three rows add up to the most common problem of all: **configuration drift**.

### Configuration drift

**Configuration drift** is when the live state of the cluster slowly moves away from the desired state written in Git. It happens through small, usually well-meant changes:

- a `kubectl scale` during a traffic spike that nobody scales back,
- a `kubectl edit` hotfix at 2 a.m. that never makes it into the repo,
- a `kubectl set image` to test something, then forgotten.

Each change is harmless on its own. But afterwards **nobody knows what is actually running**, and the next pipeline run either silently overwrites the hotfix or leaves fields it doesn't manage untouched. Either way, the YAML in Git is no longer a reliable description of the cluster.

> A push model can't detect drift. The pipeline compares nothing: it pushes and exits.

### Try it — create drift by hand

[`manifests/gitops-intro/drift-demo.yaml`](../../manifests/gitops-intro/drift-demo.yaml) is the desired state stored in Git: 2 replicas of `nginx:1.27`. Apply it once, the same way a push pipeline would:

```bash
kubectl apply -f manifests/gitops-intro/drift-demo.yaml
kubectl get deploy drift-demo        # READY 2/2
```

Now change the cluster directly with `kubectl`, bypassing Git:

```bash
kubectl scale deploy drift-demo --replicas=5
kubectl set image deploy/drift-demo nginx=nginx:1.28
```

Ask Kubernetes how the cluster now differs from the file:

```bash
kubectl diff -f manifests/gitops-intro/drift-demo.yaml
```

The diff shows `replicas: 5 → 2` and `nginx:1.28 → nginx:1.27`: the cluster has **drifted**. (`kubectl diff` exits with code `1` when it finds differences. That's expected, not an error.)

Notice what *doesn't* happen: nothing puts it back. The cluster keeps running 5 Pods of 1.28 until someone notices and runs the pipeline again. In [Getting Started with Argo CD](../argocd-getting-started/overview.md) you'll do the same experiment with Argo CD watching, and see it mark the app `OutOfSync` and undo the change.

Clean up:

```bash
kubectl delete -f manifests/gitops-intro/drift-demo.yaml
```

---

[← Install Argo CD](../environment-setup/install-argocd.md) · [↑ Contents](../../README.md) · [The GitOps Workflow →](gitops-workflow.md)
