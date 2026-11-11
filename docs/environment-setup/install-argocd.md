# Install Argo CD

> Install the Argo CD server and a matching CLI with one playbook, then log in to the UI and the CLI.

---

> ⚠️ **Installation lives in the companion Ansible tutorial.** The role is fully explained in **[ansible-tutorial → argocd Role](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/argocd.md)**. This page runs it, then explains what it put in your cluster and how to get in.

### 1. Install the `kubernetes.core` collection

The argocd role uses the `kubernetes.core.helm` module, which isn't bundled with `ansible-core`. From the root of `ansible-tutorial`:

```bash
ansible-galaxy collection install -r requirements.yml
```

> `kubernetes.core` 6.x needs **Ansible 2.16+** — newer than the rest of the Ansible tutorial needs.

### 2. Run the argocd role

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_argocd.yml
```

What it does:

- Installs the official **`argo/argo-cd` Helm chart** into the `argocd` namespace, with the chart version **pinned**, so re-running the playbook never silently upgrades Argo CD.
- Waits until every Argo CD Pod is Ready.
- Downloads the **`argocd` CLI** whose version matches the server that was just installed, so the CLI and the server can't drift apart.
- Prints the commands to fetch the admin password, but doesn't print the password itself.

> 📍 **Where things end up.** The playbook targets `hosts: control`, so everything runs on whichever machine the inventory lists under `[control]`. The **Argo CD server** always lands *in the cluster*. The inventory only decides where the Helm commands are sent from and where the **`argocd` CLI** binary is installed. This guide uses `azure.ini`, which means the CLI is installed **on the VM**. Run the `kubectl` / `argocd` commands in sections 3–4 from an SSH session on the VM.

<details>
<summary><b>Alternative: install the CLI on your own machine (<code>localhost.ini</code>)</b></summary>

`ansible/inventory/localhost.ini` puts `localhost` (with `ansible_connection=local`) in the `[control]` group. That way, Helm runs and the CLI is installed on the machine where you run Ansible:

```bash
ansible-playbook -i ansible/inventory/localhost.ini ansible/playbooks/install_kube_tools.yml   # Helm on this machine
ansible-playbook -i ansible/inventory/localhost.ini ansible/playbooks/install_argocd.yml
```

Prerequisites:

- **The cluster itself is still built on the VM.** Run `install_kubeadm.yml` with `azure.ini`, never with `localhost.ini`.
- **Helm must be installed locally.** Run `install_kube_tools.yml` with the same `localhost.ini` first.
- **Your local `~/.kube/config` must point at the remote cluster.** The role talks to the cluster through this file. Copy it from the VM and make sure its `server:` address is reachable from your machine.

</details>

### 3. What got installed

Argo CD isn't one program. It's several components, each running as its own Pod:

```bash
kubectl -n argocd get pods
```

| Pod | Role |
|-----|------|
| **argocd-server** | the API server behind both the **web UI** and the **`argocd` CLI** |
| **argocd-repo-server** | clones Git repos and renders manifests (plain YAML, Helm, Kustomize) |
| **argocd-application-controller** | the **reconcile loop**: compares the rendered manifests with the live cluster and syncs them |
| **argocd-applicationset-controller** | generates many Applications from one template (covered later) |
| **argocd-redis** | cache for the other components |
| **argocd-dex-server** | SSO / OIDC login (unused while we log in as `admin`) |
| **argocd-notifications-controller** | sends Slack/email/webhook notifications on sync events |

The heart of GitOps is the **application-controller**. The repo-server only *reads* Git, and argocd-server only *shows* the result. The chart also registers CRDs such as `applications.argoproj.io` and `appprojects.argoproj.io`. These are the objects you'll write in the next chapters:

```bash
kubectl get crd | grep argoproj.io
```

### 4. Log in

Argo CD's Service isn't exposed outside the cluster, so reach it with a port-forward. Keep this running in one terminal:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

**Web UI** — open `https://localhost:8080` and accept the self-signed certificate. Log in as `admin` with the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

> 💡 **The VM is remote?** The port-forward listens on the VM's `localhost`. Tunnel it to your laptop with `ssh -L 8080:localhost:8080 <YOUR_USER>@<YOUR_VM_IP>` and open `https://localhost:8080` locally.

**CLI** — in another terminal:

```bash
argocd login localhost:8080 --username admin --insecure \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"
```

`--insecure` skips TLS verification because the certificate is self-signed. That's fine on a lab cluster, but never use it against a real one.

**Change the password**, then delete the initial secret, as Argo CD recommends:

```bash
argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### Verify

```bash
kubectl -n argocd get pods   # every Pod Running / Ready
argocd version               # client and server versions match
argocd app list              # empty table, no error, so you're logged in
```

An empty `argocd app list` is the goal: the environment is ready, and your first Application comes in [Getting Started with Argo CD](../argocd-getting-started/overview.md).

### Uninstall

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/uninstall_argocd.yml
```

This removes the Helm release, the `argocd` namespace, and the CLI. The CRDs are kept on purpose, so your Application definitions aren't wiped by accident. The [argocd Role](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/argocd.md) page shows how to remove them too.

---

[← Set Up a Cluster](setup-cluster.md) · [↑ Contents](../../README.md) · [Introduction to GitOps →](../gitops-intro/overview.md)
