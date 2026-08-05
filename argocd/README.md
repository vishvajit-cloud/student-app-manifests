# ArgoCD Setup Guide

This guide explains how to install ArgoCD on your Kubernetes cluster and connect it to this repo, so it automatically deploys the student-app whenever Jenkins pushes a new image tag.

## What ArgoCD Actually Does

Jenkins builds the app and pushes a new Docker image, then updates `deployment.yaml` in this repo with the new image tag.

ArgoCD watches this repo. The moment it sees that change, it applies it to your Kubernetes cluster automatically. You never run `kubectl apply` by hand again.

## Step 1: Create a Namespace for ArgoCD

```bash
kubectl create namespace argocd
```

This keeps ArgoCD's own resources separate from your app's resources.

## Step 2: Install ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This single command installs everything ArgoCD needs: its server, UI, controller, and Redis cache.

Wait for all the pods to be running before moving on:

```bash
kubectl get pods -n argocd
```

## Step 3: Access the ArgoCD UI

ArgoCD's UI is not exposed to the internet by default. The simplest way to reach it is port-forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Now open your browser to:

```
https://localhost:8080
```

## Step 4: Log In

Username is always:

```
admin
```

Get the auto-generated password with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

This prints the password directly in your terminal. Copy it and log in.

## Step 5: Connect This Repo to ArgoCD

You can do this from the UI, no YAML writing required:

1. Click **+ New App** in the ArgoCD UI
2. Fill in the fields:
   - **Application Name**: `student-app`
   - **Project**: `default`
   - **Sync Policy**: `Automatic`
   - **Repository URL**: this repo's Git URL (the `student-app-manifests` repo)
   - **Revision**: `main`
   - **Path**: `.` (ex: repo root, use `.` if manifests are at the top level)
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `default`
3. Click **Create**

## Exposing ArgoCD via NodePort

By default `argocd-server` is type `ClusterIP`, only reachable inside the cluster. Follow these steps to change it to `NodePort`.

### Step 1: Open the service for editing directly in the cluster

```bash
kubectl edit svc argocd-server -n argocd
```

This opens the live service definition in your terminal's default editor (usually `vi` or `nano`).

### Step 2: Find this section inside the file

```yaml
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
```

### Step 3: Change `type: ClusterIP` to `type: NodePort`

```yaml
spec:
  type: NodePort
```

### Step 4: (Optional) Pin a fixed port instead of a random one

By default, Kubernetes assigns a random port between `30000-32767` if you just change the type and save. If you want a fixed, predictable port instead, add `nodePort:` under the `https` entry:

```yaml
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
    nodePort: 30090
```

### Step 5: Save and exit

- If it's `vi`/`vim`: press `Esc`, then type `:wq`, then `Enter`
- If it's `nano`: press `Ctrl+O`, `Enter`, then `Ctrl+X`

Kubernetes applies the change immediately on save, no restart needed.

### Step 6: Confirm the change

```bash
kubectl get svc argocd-server -n argocd
```

Under `TYPE` it should now say `NodePort`, and under `PORTS` you'll see something like `80:3xxxx/TCP,443:30090/TCP` — that second number after `443:` is what you use to access the UI: `https://<node-public-ip>:30090`

## Common Issues

**Pods stuck in Pending after install**
Your cluster may not have enough CPU/memory available. Check with `kubectl describe pod <pod-name> -n argocd` to see the exact reason.

**"repository not accessible" when adding the app**
If this is a private repo, you'll need to add credentials under Settings -> Repositories in the ArgoCD UI before creating the application.

**Password command returns nothing**
The `argocd-initial-admin-secret` is deleted the first time you log in and change your password. If you already logged in once, this secret no longer exists, that's expected, not a bug.
