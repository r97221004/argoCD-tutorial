# Accessing Argo CD

> Reach the Argo CD UI and CLI with a port-forward, log in as `admin`, and replace the initial password.

---

**Before you start:** Argo CD is installed and every Pod in the `argocd` namespace is Running, see [Installing Argo CD](installing-argocd.md). Run the commands below on the machine where the `argocd` CLI was installed, which is the VM in this guide.

### 1. Log in

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

### 2. Change the admin password

The initial password is stored **in plain text** (base64 isn't encryption) in `argocd-initial-admin-secret`, so anyone who can read Secrets in the `argocd` namespace can read it. Argo CD recommends changing it right after the first login.

**With the CLI** (you must be logged in, see above):

```bash
argocd account update-password
```

It prompts for three values:

```
*** Enter password of currently logged in user (admin): <the initial password>
*** Enter new password for user admin: <new password>
*** Confirm new password for user admin: <new password>
Password updated
Context 'localhost:8080' updated
```

> ⚠️ `--current-password` and `--new-password` flags exist for scripting, but they leave the password in your shell history. Use the interactive prompt when you type it by hand.

**Or with the UI** — **User Info** (left sidebar) → **Update Password**.

**Confirm the new password works** by logging out and back in:

```bash
argocd logout localhost:8080
argocd login localhost:8080 --username admin --insecure   # prompts for the NEW password
```

**Then delete the initial secret.** Argo CD writes it once, at first startup, and never reads it again. The real password is kept as a bcrypt hash in `argocd-secret`, so deleting it doesn't affect login:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

> Delete it only **after** you've confirmed the new password. If it's gone and you forget the new one, there's no plain-text copy left to fall back on.

<details>
<summary><b>Forgot the password? Reset it with kubectl</b></summary>

Anyone with access to the cluster can overwrite the hash in `argocd-secret`. Generate a bcrypt hash of the new password, then patch it in:

```bash
HASH=$(argocd account bcrypt --password '<NEW_PASSWORD>')
kubectl -n argocd patch secret argocd-secret \
  -p "{\"stringData\": {\"admin.password\": \"$HASH\", \"admin.passwordMtime\": \"$(date -u +%FT%TZ)\"}}"
```

> ⚠️ `argocd account bcrypt --password` puts the plain-text password in your shell history too. Clear it afterwards, or prefix the line with a space: when bash has `HISTCONTROL=ignorespace` set, lines starting with a space aren't recorded.

`admin.passwordMtime` invalidates existing sessions, so you must log in again with the new password. This is also why access to the `argocd` namespace is effectively admin access to Argo CD itself.

</details>

### Verify

```bash
argocd version               # client and server versions match
argocd app list              # empty table, no error, so you're logged in
```

An empty `argocd app list` is the goal: you can reach Argo CD from both the UI and the CLI, and it's ready for your first Application in [Core Argo CD Concepts](../argocd-core-concepts/overview.md).

---

[← Installing Argo CD](installing-argocd.md) · [↑ Contents](../../README.md) · [Core Argo CD Concepts →](../argocd-core-concepts/overview.md)
