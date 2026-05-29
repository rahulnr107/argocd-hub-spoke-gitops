# Multi-Cluster GitOps Deployment Using ArgoCD Hub-Spoke Architecture on Amazon EKS

## Project Overview

This project demonstrates centralized multi-cluster GitOps deployment using ArgoCD on Amazon EKS.

A dedicated Hub Cluster hosts the ArgoCD control plane while multiple Spoke Clusters run workloads. Applications are deployed automatically across clusters using ArgoCD ApplicationSet.

---

# Architecture

```text
GitHub Repository
        ↓
ArgoCD running in Hub Cluster
        ↓
ApplicationSet
        ↓
Automatically generates applications
        ↓
Deploys applications to Spoke Clusters
```

---

# Technologies Used

* Amazon EKS
* Kubernetes
* ArgoCD
* ApplicationSet
* AWS CLI
* kubectl
* eksctl
* GitHub

---

# Prerequisites

Install the following tools:

* AWS CLI
* kubectl
* eksctl
* ArgoCD CLI

Configure AWS credentials:

```bash
aws configure
```

---

# Step 1 — Create EKS Clusters

Create one Hub Cluster and two Spoke Clusters.

```bash
eksctl create cluster --name hub-cluster --region us-west-1

eksctl create cluster --name spoke-cluster-1 --region us-west-1

eksctl create cluster --name spoke-cluster-2 --region us-west-1
```

Verify cluster contexts:

```bash
kubectl config get-contexts
```

Switch to Hub Cluster:

```bash
kubectl config use-context <hub-cluster-context>
```

Verify current context:

```bash
kubectl config current-context
```

---

# Step 2 — Install ArgoCD in Hub Cluster

Create namespace:

```bash
kubectl create namespace argocd
```

Install ArgoCD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify pods:

```bash
kubectl get pods -n argocd
```

---

# Step 3 — Enable ArgoCD Insecure Mode (HTTP)

For learning purposes, ArgoCD was configured to run in HTTP mode instead of HTTPS.

Edit ConfigMap:

```bash
kubectl edit configmap argocd-cmd-params-cm -n argocd
```

Add:

```yaml
server.insecure: "true"
```

Restart ArgoCD server:

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

---

# Step 4 — Expose ArgoCD Using NodePort

Check services:

```bash
kubectl get svc -n argocd
```

Edit service:

```bash
kubectl edit svc argocd-server -n argocd
```

Change:

```yaml
type: ClusterIP
```

To:

```yaml
type: NodePort
```

Verify:

```bash
kubectl get svc -n argocd
```

Open the NodePort in the EC2 security group.

Access ArgoCD UI:

```text
http://<EC2-PUBLIC-IP>:<NODEPORT>
```

---

# Step 5 — Retrieve ArgoCD Admin Password

Get secrets:

```bash
kubectl get secrets -n argocd
```

Get admin password:

```bash
kubectl edit secret argocd-initial-admin-secret -n argocd
```

Decode password:

```bash
echo <password> | base64 --decode
```

Login:

```text
Username: admin
Password: decoded password
```

---

# Step 6 — Login to ArgoCD CLI

```bash
argocd login <EC2-PUBLIC-IP>:<NODEPORT> --insecure
```

Use the same username and password used for UI login.

---

# Step 7 — Register Spoke Clusters to ArgoCD

Get contexts:

```bash
kubectl config get-contexts
```

Add Spoke Cluster 1:

```bash
argocd cluster add <spoke-cluster-1-context> --server <EC2-PUBLIC-IP>:<NODEPORT> --insecure
```

Add Spoke Cluster 2:

```bash
argocd cluster add <spoke-cluster-2-context> --server <EC2-PUBLIC-IP>:<NODEPORT> --insecure
```

Verify registered clusters:

```bash
argocd cluster list
```

---

# Step 8 — Verify ApplicationSet Controller

```bash
kubectl get pods -n argocd
```

Verify:

```text
argocd-applicationset-controller
```

---

# Step 9 — Create ApplicationSet

Create file:

```text
applicationset.yaml
```

Add:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: guestbook-appset
  namespace: argocd

spec:
  generators:
  - clusters: {}

  template:
    metadata:
      name: '{{name}}-guestbook'

    spec:
      project: default

      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook

      destination:
        server: '{{server}}'
        namespace: default

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

Apply:

```bash
kubectl apply -f applicationset.yaml
```

---

# Step 10 — Verify Application Deployment

Switch to Spoke Cluster:

```bash
kubectl config use-context <spoke-cluster-1-context>
```

Verify resources:

```bash
kubectl get all
```

Repeat for other spoke clusters.

---

# GitOps Repository Structure

```text
gitops-repo/
│
├── applicationsets/
│   └── applicationset.yaml
│
├── apps/
│   └── guestbook/
│       ├── deployment.yaml
│       └── service.yaml
```

---

# Conclusion

This project demonstrates scalable multi-cluster GitOps deployment using ArgoCD Hub-Spoke architecture on Amazon EKS.

Applications are automatically generated and deployed across multiple clusters using ApplicationSet, improving scalability and centralized deployment management.
