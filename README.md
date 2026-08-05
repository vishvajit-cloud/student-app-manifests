# student-app-manifests

This is the **GitOps repo** for the `student-app` project.

It only contains Kubernetes manifests (`deployment.yaml`, `service.yaml`).
It does **not** contain any application source code.

## How it works
1. Your app's Jenkins pipeline (in the `student-app` repo) builds, tests, scans,
   builds a Docker image, and pushes it to DockerHub.
2. As the last stage, Jenkins clones **this** repo, updates the `image:` tag
   inside `deployment.yaml` (v1 -> v2 -> v3 ...), and pushes the change back
   to this repo.
3. **ArgoCD** is configured to watch this repo. As soon as it detects the
   commit, it automatically syncs the change to your Kubernetes cluster.

## Before first use
Replace the placeholders in `deployment.yaml`:
- `<DOCKERHUB_USERNAME>` — your DockerHub username (Jenkins will keep the
  tag updated automatically, but the username stays fixed).
- `<DB_HOST>`, `<DB_NAME>`, `<DB_USERNAME>`, `<DB_PASSWORD>` — your MySQL
  connection details (better long-term: move these into a Kubernetes
  Secret/ConfigMap instead of plain env values).

## Setting up ArgoCD to watch this repo
In ArgoCD, create an Application pointing to:
- Repo URL: this repo's Git URL
- Path: `.` (repo root, or wherever you place these files)
- Destination: your target cluster/namespace
- Sync policy: automated (so it deploys as soon as Jenkins pushes)
