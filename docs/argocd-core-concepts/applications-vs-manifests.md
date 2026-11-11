# Applications vs. Kubernetes Manifests

> An Application doesn't *contain* your app. It *points at* the manifests that do.

---

The most common beginner mix-up: "I created the Application, so where's my Deployment inside it?" It isn't inside. There are **two separate layers**:

```
  argocd namespace              Git (<GITOPS_REPO>)                    guestbook namespace
┌─────────────────────┐       ┌───────────────────────────────┐      ┌──────────────────────┐
│ Application         │ reads │ guestbook/                    │ apply│ Deployment           │
│   guestbook         │──────▶│   guestbook-ui-deployment.yaml│─────▶│ Service              │
│ (where from, to)    │       │   guestbook-ui-svc.yaml       │      │ Pods                 │
└─────────────────────┘       └───────────────────────────────┘      └──────────────────────┘
   the pointer                  the manifests                         what actually runs
```

| | Application | Kubernetes manifests |
|---|---|---|
| What it is | Instructions **for Argo CD** | Instructions **for Kubernetes** |
| Kind | `Application` (an Argo CD CRD) | `Deployment`, `Service`, `ConfigMap`, … |
| Lives in | the `argocd` namespace | your app's namespace (`guestbook`) |
| Stored in Git at | this tutorial: `manifests/argocd-core-concepts/application.yaml` | the config repo `<GITOPS_REPO>`: `guestbook/` |
| You change it when… | the app moves to another repo, path, cluster or namespace | the app itself changes: new image, more replicas, … |

In short: **the Application says *where*, the manifests say *what*.**

### Consequences worth knowing

- **Day-to-day changes go to the manifests, not the Application.** To deploy a new image you edit `guestbook-ui-deployment.yaml` and push. The Application stays untouched.
- **You never `kubectl apply` the manifests yourself.** Argo CD does that. If you do it by hand, it's the drift you saw in [the push model](../gitops-intro/push-model.md).
- **The manifests are plain Kubernetes YAML.** The same files would work with `kubectl apply -f`. Argo CD adds no special syntax to them.
- **Deleting an Application** with `argocd app delete` also deletes the resources it created. That's how you'll clean up after the lab.

See both layers once your first app is running:

```bash
kubectl get applications -n argocd   # the pointer
kubectl get deploy,svc -n guestbook   # what it deployed
```

---

[← The Application CRD](application-crd.md) · [↑ Contents](../../README.md) · [Lab: Deploying Your First Application →](first-application.md)
