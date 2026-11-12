# GitOps with Argo CD — Lab Report

This lab sets up Argo CD on a Kind Kubernetes cluster and uses it to deploy a simple web app from GitHub. It then tests three core GitOps behaviours: Git as the single source of truth, self-healing, and automatic pruning.

**Repository:** [`C-gyeltshen/dso202-gitops-demo`](https://github.com/C-gyeltshen/dso202-gitops-demo)

---

## Table of Contents

1. [Environment Setup](#1-environment-setup)
2. [Install Argo CD](#2-install-argo-cd)
3. [Access the Argo CD UI](#3-access-the-argo-cd-ui)
4. [Prepare the GitHub Repository](#4-prepare-the-github-repository)
5. [Register the Application in Argo CD](#5-register-the-application-in-argo-cd)
6. [Experiment 1 — Git as the Single Source of Truth](#6-experiment-1--git-as-the-single-source-of-truth)
7. [Experiment 2 — Configuration Drift & Self-Healing](#7-experiment-2--configuration-drift--self-healing)
8. [Experiment 3 — Automatic Pruning](#8-experiment-3--automatic-pruning)
9. [Issues Encountered](#9-issues-encountered)

---

## 1. Environment Setup

The lab reuses the existing 3-node Kind cluster **`dso202`** (1 control plane, 2 worker nodes).

Check that the cluster containers are running:

```bash
docker ps
```

![Docker containers](./evidence/1.png)

Check that all nodes are `Ready`:

```bash
kubectl get nodes
```

![Cluster nodes](./evidence/2.png)

---

## 2. Install Argo CD

Argo CD runs in its own dedicated namespace, `argocd`.

```bash
# Create the namespace
kubectl create namespace argocd

# Install the official stable Argo CD manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

![Create namespace](./evidence/3.png)
![Install Argo CD](./evidence/4.png)

Wait for the Argo CD pods to become ready:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

![Argo CD pods](./evidence/5.png)

---

## 3. Access the Argo CD UI

### 3.1 Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

> The password is not recorded here. Credentials should never be committed to Git.

### 3.2 Port-forward the Argo CD server

Run this in a separate terminal and keep it open:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

![Port forward](./evidence/7.png)

### 3.3 Log in

1. Open **https://localhost:8080**.
2. Accept the self-signed certificate warning (**Advanced → Proceed to localhost**).
3. Log in with username `admin` and the password from step 3.1.

![Argo CD login](./evidence/8.png)

---

## 4. Prepare the GitHub Repository

### 4.1 Create and clone the repo

A public repository named `dso202-gitops-demo` was created on GitHub with a README so the `main` branch is initialised.

![GitHub repo](./evidence/9.png)

```bash
git clone https://github.com/C-gyeltshen/dso202-gitops-demo.git
cd dso202-gitops-demo
```

![Clone repo](./evidence/10.png)

### 4.2 Add the Kubernetes manifests

```bash
mkdir manifests
```

![Create manifests folder](./evidence/11.png)

**`manifests/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deploy
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
```

![deployment.yaml](./evidence/12.png)

**`manifests/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

![service.yaml](./evidence/13.png)

### 4.3 Commit and push

```bash
git add .
git commit -m "Initial deployment with 2 replicas"
git push origin main
```

![Push to GitHub](./evidence/14.png)

---

## 5. Register the Application in Argo CD

The Argo CD `Application` is defined declaratively in `argocd-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp
  namespace: argocd            # Applications always live in the argocd namespace
spec:
  project: default
  source:
    repoURL: https://github.com/C-gyeltshen/dso202-gitops-demo.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-app        # Where the app's resources are deployed
  syncPolicy:
    automated:
      prune: true              # Delete resources removed from Git
      selfHeal: true           # Revert manual changes in the cluster
    syncOptions:
      - CreateNamespace=true   # Create demo-app if it does not exist
```

Apply it:

```bash
kubectl apply -f argocd-app.yaml
```

![Apply Application](./evidence/15.png)
![Application in Argo CD UI](./evidence/16.png)

A `manifests/namespace.yaml` was also added so the `demo-app` namespace is defined explicitly in Git (see [Issues Encountered](#9-issues-encountered)):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
```

![namespace.yaml](./evidence/17.png)

Verify the pods are running across the worker nodes:

```bash
kubectl get pods -n demo-app -o wide
```

![Pods in demo-app](./evidence/18.png)

**Where each resource lives:**

| Resource | Namespace |
| :--- | :--- |
| Application `my-webapp` | `argocd` |
| Deployment `webapp-deploy` | `demo-app` |
| Service `webapp-svc` | `demo-app` |

---

## 6. Experiment 1 — Git as the Single Source of Truth

**Goal:** scale the app from 2 to 4 replicas by changing Git only.

1. In `manifests/deployment.yaml`, change `replicas: 2` to `replicas: 4`.

   ![Edit replicas](./evidence/19.png)

2. Commit and push to `main`.

   ![Commit change](./evidence/20.png)

3. In the Argo CD UI, click **Refresh** (or wait up to 3 minutes for polling).

**Result:** Argo CD detects the new commit and creates 2 additional pods, bringing the total to 4.

---

## 7. Experiment 2 — Configuration Drift & Self-Healing

**Goal:** show that manual changes made outside Git are reverted.

Manually scale the deployment down, bypassing Git:

```bash
kubectl scale deployment webapp-deploy -n demo-app --replicas=1
```

![Manual scale and self-heal](./evidence/21.png)

**Result:** the app briefly shows `OutOfSync`. Because `selfHeal: true` is set, Argo CD restores the deployment to the 4 replicas defined in Git within seconds.

---

## 8. Experiment 3 — Automatic Pruning

**Goal:** show that deleting a manifest from Git deletes the resource from the cluster.

1. Delete `manifests/service.yaml` from the repo and commit the deletion.
2. Click **Refresh** in the Argo CD UI.
3. Because `prune: true` is set, Argo CD deletes `webapp-svc` from the `demo-app` namespace.
4. *(Optional)* Restore the Service with `git revert HEAD` and push.

---

## 9. Issues Encountered

### Repository URL mismatch

**Symptom:** `kubectl get pods -n demo-app` returned `No resources found`, and the application showed `Sync Status: Unknown`.

```bash
kubectl describe application my-webapp -n argocd
```

reported:

```text
ComparisonError: failed to list refs: authentication required: Repository not found.
```

**Cause:** the lab guide names the repository `dso202-gitops-demo`, but its clone command and `repoURL` use `k8s-gitops-demo`. Argo CD was pointing at a repository that does not exist.

**Fix:** update `repoURL` in `argocd-app.yaml` to the real repository, then re-apply and force a refresh:

```bash
sed -i '' 's/k8s-gitops-demo/dso202-gitops-demo/' argocd-app.yaml
kubectl apply -f argocd-app.yaml
kubectl annotate application my-webapp -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### The `demo-app` namespace

The guide never runs `kubectl create namespace demo-app`. This is intentional: `CreateNamespace=true` in the Application's `syncOptions` makes Argo CD create the namespace on its first successful sync. The namespace only appeared to be missing because the sync was failing (see above). `manifests/namespace.yaml` was added to make the namespace explicit in Git. Note that with `prune: true`, deleting this file from Git would delete the namespace and everything in it.

