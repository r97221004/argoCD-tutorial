# Argo CD in Practice

> _Learn GitOps with Argo CD and Argo Rollouts by deploying real apps from Git, one concept at a time._

![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.30%2B-326CE5?logo=kubernetes&logoColor=white)
[![License: MIT](https://img.shields.io/badge/License-MIT-green)](LICENSE)

> **Problem**: _TODO_
>
> **Solution**: _TODO_

## What You'll Learn

_TODO_

## Prerequisites

- **Kubernetes basics** — Pods, Deployments, Services, `kubectl`. New to them? Start with [Kubernetes in Practice](https://github.com/r97221004/k8s-tutorial).
- **A control machine with Ansible** and a clone of [ansible-tutorial](https://github.com/r97221004/ansible-tutorial) — every install in this guide is an Ansible role.
- **A target Linux VM** (Ubuntu 22.04, 2 vCPU / 4 GB RAM / 20 GB disk) reachable over SSH with passwordless sudo.

Full walk-through in [Environment Setup](docs/environment-setup/setup-cluster.md).

## Table of Contents

**Part 1 — Environment Setup**

- [**Set Up a Cluster**](docs/environment-setup/setup-cluster.md) — a single-node kubeadm cluster plus k9s and Helm, via Ansible.

**Part 2 — Introduction to GitOps**

- [**The Push Model and Its Drawbacks**](docs/gitops-intro/push-model.md) — why CI-driven `kubectl apply` leads to configuration drift.
- [**The GitOps Workflow**](docs/gitops-intro/gitops-workflow.md) — pull instead of push: Argo CD reconciles the cluster from Git.
- [**The GitOps Principles**](docs/gitops-intro/gitops-principles.md) — the four OpenGitOps principles and how Argo CD meets each.

**Part 3 — [Argo CD] Getting Started with Argo CD**

- [**Installing Argo CD**](docs/argocd-getting-started/installing-argocd.md) — the Argo CD Helm chart and a matching CLI via Ansible, and what each component does.
- [**Accessing Argo CD**](docs/argocd-getting-started/accessing-argocd.md) — reach the UI and CLI with a port-forward, log in, and change the admin password.

**Part 4 — [Argo CD] Core Argo CD Concepts**

- [**The Argo CD Architecture**](docs/argocd-core-concepts/architecture.md) — the three core components and what breaks when each is down.
- [**The Application CRD**](docs/argocd-core-concepts/application-crd.md) — where from, where to, and how: the resource that tells Argo CD what to deploy. *Lab:* fork, apply an Application, and sync.
- [**Applications vs. Kubernetes Manifests**](docs/argocd-core-concepts/applications-vs-manifests.md) — the pointer vs. what it points at.
- [**Sync and Health Checks**](docs/argocd-core-concepts/sync-and-health.md) — two independent statuses, refresh vs. sync, and sync policies. *Lab:* Git changes, drift, and self-heal in action.
