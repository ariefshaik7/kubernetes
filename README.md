# Kubernetes Cluster Setup using Kubeadm

Welcome to the Kubernetes setup project! This repository demonstrates how to manually set up a Kubernetes cluster using `kubeadm` and access it locally for management and testing.

---

##  What is Kubeadm?

[`kubeadm`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is the **official Kubernetes tool** designed to help you **bootstrap a production-ready Kubernetes cluster** quickly and easily.

It handles:

- Cluster initialization
- Certificate generation
- API server bootstrapping
- Token-based worker node joining

Unlike managed Kubernetes services (like AKS, EKS, GKE), `kubeadm` gives you **full control** over every component of your cluster, making it ideal for learning, testing, or advanced production customization.

---


## Project Structure

This repository is structured into two main folders:

- [`kubeadm-k8s/`](./kubeadm-k8s):  
  This directory contains the complete step-by-step guide to setting up a multi-node Kubernetes cluster manually using `kubeadm`. It includes:
  - Control-plane and worker node configuration
  - Required networking setup
  - Installation of dependencies like `containerd`, `kubectl`, `kubelet`, and `kubeadm`
  - Instructions for joining nodes and verifying cluster health

- [`kube-config/`](./kube-config):  
  This folder contains instructions for securely accessing the Kubernetes cluster *from your local machine*. It includes:
  - How to install `kubectl` locally
  - How to copy your kubeconfig file from the master VM
  - How to enable local DNS resolution
  - How to switch between clusters and contexts

---


## Who is This For?

This project is ideal for:

- Students and professionals learning Kubernetes administration from scratch
- DevOps engineers who want to understand the internals of Kubernetes setup
- Anyone preparing for Kubernetes certification exams (CKA, CKAD)
- Cloud-native developers who want to build a cluster in a custom environment

---

## Prerequisites

Before you dive in, make sure you have:

- Three Ubuntu VMs (1 master + 2 workers) with public IPs
- A way to SSH into each machine
- Basic networking knowledge (firewall rules, IPs, DNS)
- Patience and curiosity!

---

## Getting Started

1. Go to [`kubeadm-k8s/`](./kubeadm-k8s) and follow the setup instructions to build your cluster.
2. Then head to [`kube-config/`](./kube-config) to configure secure local access via `kubectl`.
3. Test your cluster and deploy your first app!

---

## Feedback

If you find this project helpful or have suggestions, feel free to open an issue or submit a pull request. Contributions are welcome!

---
