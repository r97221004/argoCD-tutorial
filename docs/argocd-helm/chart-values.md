# Overriding Chart Values

> A chart's `values.yaml` holds the defaults. The Application says which of them to change, and the change lives in Git with everything else.

---

With plain `helm` you'd pass `-f values-production.yaml` or `--set replicaCount=3`. With Argo CD the same choices go into the Application, under `spec.source.helm`, so they're declarative and versioned instead of typed on a command line.

### Four ways to set values

| Field in `spec.source.helm` | Where the values live | Like Helm's | GitOps-friendly? |
|-----------------------------|-----------------------|-------------|------------------|
| `valueFiles` | YAML files **next to the chart**, e.g. `values-production.yaml` | `-f file` | ✅ in Git, reviewed like code |
| `valuesObject` | **inline** in the Application, as YAML | `-f` with an inline file | ✅ if the Application is in Git |
| `values` | inline, as one YAML **string** | `-f` | ✅ older form of `valuesObject`; prefer `valuesObject` |
| `parameters` | inline, as `name` / `value` pairs | `--set` | ⚠️ fine in YAML, but `argocd app set --helm-set` writes them **only in the cluster** |

`valueFiles` paths are relative to the chart folder (`path`), e.g. `valueFiles: [values-production.yaml]` for `helm-guestbook/values-production.yaml`. The inline forms look like this:

```yaml
  source:
    repoURL: https://github.com/<YOUR_GITHUB_USER>/argocd-apps.git
    targetRevision: HEAD
    path: helm-guestbook
    helm:
      valuesObject:
        replicaCount: 3
      parameters:
        - name: replicaCount
          value: "2"
```

### Which value wins?

When the same key is set in several places, the more specific one wins:

| Precedence | Source |
|------------|--------|
| 1 (lowest) | the chart's own `values.yaml` |
| 2 | `valueFiles`, later files over earlier ones |
| 3 | `values` / `valuesObject` (`valuesObject` wins if both are set) |
| 4 (highest) | `parameters` |

So in the example above, `replicaCount` ends up **2**: `parameters` overrides the 3 from `valuesObject`, which overrides the 1 in `values.yaml`. A values file listed in `valueFiles` would slot in between `values.yaml` and `valuesObject`. The lab walks through exactly that example.

> **Values for a public chart.** A `valueFiles` path must be inside the chart's own source, and you can't add files to someone else's Helm repository. To keep a public chart's values in *your* Git repo, give the Application two `sources`: the chart, and your repo marked `ref: values`, then point at the file as `$values/…`:
>
> ```yaml
>   sources:
>     - repoURL: https://kubernetes-sigs.github.io/headlamp/
>       chart: headlamp
>       targetRevision: 0.46.0
>       helm:
>         valueFiles:
>           - $values/headlamp/values.yaml
>     - repoURL: https://github.com/<YOUR_GITHUB_USER>/argocd-apps.git
>       targetRevision: HEAD
>       ref: values
> ```
>
> For a few keys, `valuesObject` on the chart source is simpler.

### Lab: Set Chart Values

> Override `helm-guestbook`'s replica count inline, then override it again with a parameter, and watch the precedence rules decide.

<details>
<summary><b>Show the lab steps</b></summary>

**Goal:** change a chart's values only through the Application, review each change as a diff, and see which value wins.

**Before you start:** `helm-guestbook` from [Lab: Deploy a Helm Chart from Git](helm-charts.md#lab-deploy-a-helm-chart-from-git) is `Synced` and `Healthy` with 1 replica. All edits are to [`manifests/argocd-helm/helm-guestbook.yaml`](../../manifests/argocd-helm/helm-guestbook.yaml) in the tutorial clone.

```bash
cd ~/argoCD-tutorial   # wherever you cloned the tutorial
```

#### 1. Override a value inline

Add a `helm` block under `source`, after `path`:

```yaml
    path: helm-guestbook
    helm:
      valuesObject:
        replicaCount: 3
```

Apply it and compare before syncing:

```bash
kubectl apply -f manifests/argocd-helm/helm-guestbook.yaml
argocd app diff helm-guestbook       # replicas: 1 → 3
argocd app sync helm-guestbook
kubectl get deploy helm-guestbook -n helm-guestbook   # READY 3/3
```

The 3 from `valuesObject` is layered over the chart's `values.yaml`. Every key you didn't set, like the image or the Service type, keeps its default.

#### 2. Override it again with a parameter

Add `parameters` next to `valuesObject`. Parameter values are always strings, so quote the number:

```yaml
    helm:
      valuesObject:
        replicaCount: 3
      parameters:
        - name: replicaCount
          value: "2"
```

```bash
kubectl apply -f manifests/argocd-helm/helm-guestbook.yaml
argocd app diff helm-guestbook       # replicas: 3 → 2
argocd app sync helm-guestbook
kubectl get deploy helm-guestbook -n helm-guestbook   # READY 2/2
```

`valuesObject` still says 3, but `parameters` ranks higher, so 2 wins. Here the parameter is in the Application file, so it's still declarative. Set the same thing with `argocd app set --helm-set` and it would exist only in the cluster.

#### 3. See the values Argo CD used

```bash
argocd app manifests helm-guestbook | grep replicas   # replicas: 2
```

In the UI, open **helm-guestbook** → **App Details** → **Parameters**: it lists every value the chart was rendered with. When a result surprises you, this is where to look first.

#### Checkpoint

| You did | Which means |
|---------|-------------|
| `valuesObject: {replicaCount: 3}` | inline values override the chart's `values.yaml`; keys you don't set keep their defaults |
| `parameters: replicaCount=2` | parameters beat every other way of setting values |
| `argocd app diff` before each sync | every value change was reviewed before it reached the cluster |

#### Clean up

Delete both Helm apps from this part. `argocd app delete` also deletes what they deployed:

```bash
argocd app delete helm-guestbook --yes
argocd app delete headlamp --yes      # if you deployed it in the previous lab
kubectl delete ns helm-guestbook headlamp --ignore-not-found
```

Undo your edits to the Application files, so the tutorial clone is clean again (this also resets the GitHub account you filled in):

```bash
git checkout -- manifests/argocd-helm/
```

</details>

---

[← Deploying Public Helm Charts](public-helm-charts.md) · [↑ Contents](../../README.md)
