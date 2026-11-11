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
- [**Install Argo CD**](docs/environment-setup/install-argocd.md) — the Argo CD Helm chart and a matching CLI, via Ansible; log in to the UI and CLI.

**Part 2 — Introduction to GitOps**

- [**Introduction to GitOps**](docs/gitops-intro/overview.md) — _TODO_

**Part 3 — [Argo CD] Getting Started with Argo CD**

- [**Getting Started with Argo CD**](docs/argocd-getting-started/overview.md) — _TODO_

**Part 4 — [Argo CD] Core Argo CD Concepts**

- [**Core Argo CD Concepts**](docs/argocd-core-concepts/overview.md) — _TODO_
