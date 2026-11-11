# Set Up a Cluster

> Argo CD runs *inside* Kubernetes — so first you need a cluster, provisioned with the companion Ansible tutorial.

---

> ⚠️ **Build the cluster before starting this chapter.** The install steps aren't repeated here — they're fully covered in the companion Ansible tutorial, which provisions a single-node cluster over SSH. This page only lists which playbooks to run, in which order, and how to confirm the result.

Argo CD is not a tool you run on your laptop: it's a set of controllers that live **in the cluster** and pull manifests from Git. So the environment for this course is the same single-node lab built in [Kubernetes in Practice](https://github.com/r97221004/k8s-tutorial), plus Helm to install Argo CD with.

### What you need

- **A control machine** with Ansible and a clone of [r97221004/ansible-tutorial](https://github.com/r97221004/ansible-tutorial) — see its [Install Ansible](https://github.com/r97221004/ansible-tutorial/blob/master/docs/getting-started/install-ansible.md).
- **A target Linux VM** (Ubuntu 22.04) reachable over SSH, with passwordless sudo — see [Switch to SSH Connection](https://github.com/r97221004/ansible-tutorial/blob/master/docs/fundamentals/ssh-connection.md). A comfortable size is **2 vCPU, 4 GB RAM, 20 GB disk**; Argo CD adds about 7 Pods on top of the control plane.
- **The VM in the inventory** — edit `ansible/inventory/azure.ini`:

  ```ini
  [control]
  azure-vm ansible_host=<YOUR_VM_IP> ansible_user=<YOUR_USER> ansible_connection=ssh
  ```

Check that Ansible can reach it:

```bash
ansible control -i ansible/inventory/azure.ini -m ping
```

### 1. Kubernetes — the kubeadm role

Run from the root of the `ansible-tutorial` repo:

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kubeadm.yml
```

This builds a vanilla, single-node Kubernetes cluster (containerd, kubelet/kubeadm/kubectl, flannel) and copies the admin kubeconfig to `~/.kube/config`. Every step is explained in **[ansible-tutorial → kubeadm Role](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/kubeadm.md)**, and what each component *is* in **[k8s-tutorial → Set Up a Cluster (kubeadm)](https://github.com/r97221004/k8s-tutorial/blob/master/docs/getting-started/setup-kubeadm.md)**.

> **Fast lane — k3s.** Every playbook in this chapter also works on k3s: swap in `install_k3s.yml`. k3s comes up faster but hides more; kubeadm stays the reference cluster for this guide. Comparison in [Choosing k3s vs kubeadm](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/k3s-vs-kubeadm.md).

### 2. k9s and Helm — the kube_tools role

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kube_tools.yml
```

- **Helm** — required: Argo CD is installed from its official Helm chart in [Installing Argo CD](../argocd-getting-started/installing-argocd.md).
- **k9s** — optional but recommended: you'll use it to *watch* Argo CD create, heal, and prune resources. Details in **[ansible-tutorial → kube_tools Role](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/kube-tools.md)**.

### Verify your cluster

SSH into the VM and confirm:

```bash
kubectl get nodes                 # one node, STATUS Ready
kubectl get pods -n kube-system   # control plane, CoreDNS, kube-proxy, flannel all Running
helm version                      # Helm is installed
k9s version                       # k9s is installed
```

If any of these fail, fix the cluster first — see the Ansible tutorial's [Troubleshooting](https://github.com/r97221004/ansible-tutorial/blob/master/docs/operating/troubleshooting.md). Argo CD can't be healthier than the cluster it runs on.

---

[↑ Contents](../../README.md) · [The Push Model and Its Drawbacks →](../gitops-intro/push-model.md)
