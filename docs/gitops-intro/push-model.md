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
    lineColor: '#94A3B8'
    edgeLabelBackground: '#475569'
  themeCSS: |
    .edgeLabel, .edgeLabel p, .edgeLabel span { color: #FFFFFF !important; }
---
flowchart LR
    dev(["👩‍💻 developer"])
    git["📚 app repo<br/>code + YAML / chart"]

    subgraph ci["⚙️ CI/CD pipeline · outside the cluster"]
        direction TB
        build["🔨 build job<br/>test · build · push image"]
        deploy["📤 deploy job<br/>auto or manual approval<br/>helm upgrade --install<br/>🔑 holds cluster credentials"]
        build --> deploy
    end

    reg[("🗃️ image registry")]

    subgraph cluster["☸️ Kubernetes cluster"]
        direction TB
        api["🚪 kube-apiserver"]
        live["🚀 Deployment · Pods"]
        api --> live
    end

    ops(["🧑‍🔧 someone with kubectl"])

    dev -->|git push| git
    git -->|triggers| build
    build -->|push image| reg
    deploy ==>|"PUSH · once per run"| api
    ops -. "kubectl edit / scale<br/>(not in Git)" .-> api
    live -. pulls image .-> reg

    classDef ctrl fill:#0F172A,stroke:#2563EB,stroke-width:2px,color:#FFFFFF
    classDef eng fill:#D97706,stroke:#92400E,stroke-width:3px,color:#FFFFFF
    classDef tgt fill:#0F172A,stroke:#16A34A,stroke-width:2px,color:#FFFFFF
    class dev,ops,git ctrl
    class deploy eng
    class build,reg,api,live tgt
    linkStyle 5 stroke:#D97706,stroke-width:3px
    style ci fill:#000000,stroke:#94A3B8,color:#FFFFFF,fillStyle:solid
    style cluster fill:#1E293B,stroke:#334155,color:#F1F5F9,fillStyle:solid
```

This is the same **one-shot** habit as running `kubectl apply` by hand. It's just automated: something outside the cluster sends the change once, and the job is considered done when the command returns.

> **Packaging the chart too.** Some pipelines go one step further: the build job also packages the Helm chart (`helm package` → a `.tgz`) and pushes it to an OCI registry next to the image, and the deploy job installs that exact version (`helm upgrade --install myapp oci://<registry>/charts/myapp --version 1.4.2`). The chart is now versioned like the image, but the model doesn't change: the pipeline still pushes into the cluster with its own credentials.

### The drawbacks

| Problem | Why |
|---------|-----|
| 🔑 **CI holds the keys to the cluster** | The pipeline needs a kubeconfig with write access. Anyone who can read it, such as another CI job or a leaked log, can change production. |
| 👀 **Nobody watches after the deploy** | The job ends when the deploy command returns. Even `helm upgrade --wait` only checks once, at deploy time; whether the Pods stay healthy, or get changed later, nothing checks. |
| 🕵️ **No record of who changed what** | A `kubectl` change leaves no commit, no review and no reason. Git shows what the pipeline deployed, not what's actually running. |
| ⏪ **Rollback means re-deploying** | Undoing a bad release means re-running an old pipeline under pressure, or a quick `helm rollback` that Git never hears about, which is drift again. |
| 🔀 **The cluster drifts away from Git** | Anyone with `kubectl` can change the cluster directly, and nothing changes it back. |

The last one has a name: **configuration drift**.

### Configuration drift

**Configuration drift** is when the live cluster slowly stops matching the YAML in Git. It comes from small, well-meant changes: a `kubectl scale` during a traffic spike or a `kubectl edit` hotfix at 2 a.m. that never makes it back into the repo.

Each one is harmless alone. Together they mean **Git no longer tells you what's actually running**, and a push model can't notice: the pipeline pushes and exits.

With several environments it gets worse: dev, staging and prod each drift in their own way, so a release that worked in staging may not behave the same in prod.

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

Notice what *doesn't* happen: nothing puts it back. The cluster keeps running 5 Pods of 1.28 until someone notices and runs the pipeline again. In the [sync lab](../argocd-core-concepts/sync-and-health.md#lab-the-sync-process) you'll do the same experiment with Argo CD watching, and see it mark the app `OutOfSync` and undo the change.

Clean up:

```bash
kubectl delete -f manifests/gitops-intro/drift-demo.yaml
```

---

[← Set Up a Cluster](../environment-setup/setup-cluster.md) · [↑ Contents](../../README.md) · [The GitOps Workflow →](gitops-workflow.md)
