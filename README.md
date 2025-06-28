# Kube-OVH GitOps Repository

This repository contains the configuration for a Kubernetes cluster managed with a GitOps workflow using ArgoCD. It follows the app-of-apps pattern to manage cluster-wide addons and applications.

## Overview

The core idea is to have a single source of truth for the cluster's desired state, which is this Git repository. ArgoCD continuously monitors this repository and automatically applies any changes to the cluster, ensuring the live state matches the state defined here.

## Features

*   **GitOps with ArgoCD:** All cluster configuration is managed declaratively in this repository.
*   **App-of-Apps Pattern:** A root ArgoCD application (`app-of-apps`) manages other applications, providing a single entry point for bootstrapping the cluster.
*   **Automated Addon Management:** An `ApplicationSet` automatically discovers and deploys cluster addons (like cert-manager, ingress-nginx, etc.) from the `addons/` directory.
*   **Tag-Based Deployments:** Deployments are triggered by Git tags, allowing for versioned and controlled rollouts instead of tracking the `main` branch.

## How it Works

1.  **Root Application (`app-of-apps.yaml`):** This is the first thing you apply to the cluster. It's an ArgoCD Application that points to this Git repository. Its main job is to deploy other ArgoCD resources found in the `/argocd` directory. You must manually update the `targetRevision` in this file to a specific Git tag to control the base version of your cluster setup.

2.  **Addon ApplicationSet (`argocd/addons-applicationset.yaml`):** This `ApplicationSet` is deployed by the root application. It uses a Git generator to scan this repository for new Git tags matching the pattern `v.*`. For each matching tag, it generates ArgoCD `Application` resources for each addon defined in the `addons/` directory. This means when you push a new `v*` tag, all addons are automatically updated to the version of the code specified by that tag.

3.  **Addons:** Each subdirectory in the `addons/` directory represents a cluster addon (e.g., `cert-manager`, `ingress-nginx`). These are Helm charts that the `ApplicationSet` will deploy.

## Prerequisites

*   A running Kubernetes cluster.
*   `kubectl` installed and configured to connect to your cluster.
*   ArgoCD installed in the cluster.

## Deployment Steps

### 1. Install ArgoCD

If you haven't already, install ArgoCD on your cluster. You can follow the official ArgoCD documentation:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Create and Push a Git Tag

Since deployments are based on tags, you need to create and push a tag for your initial deployment.

```bash
# Create a tag (e.g., v1.0.0)
git tag v1.0.0

# Push the tag to your remote repository
git push origin v1.0.0
```

### 3. Update and Apply the Root Application

Before applying the root application, ensure the `targetRevision` in `app-of-apps.yaml` points to the Git tag you just created.

For example, if you created the tag `v1.0.0`, your `app-of-apps.yaml` should look like this:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  # ...
spec:
  # ...
  source:
    repoURL: https://github.com/leogout/kube-ovh.git # Change this to your fork
    targetRevision: v1.0.0 # <-- Make sure this matches your tag
    path: "argocd"
  # ...
```

Now, apply the root application to your cluster:

```bash
kubectl apply -f app-of-apps.yaml
```

### 5. Verify the Deployment

Once you apply the `app-of-apps.yaml` file, ArgoCD will start the synchronization process. It will first deploy the `ApplicationSet` from the `/argocd` directory. Then, the `ApplicationSet` will detect your new tag and create applications for all the addons in the `addons/` directory.

You can check the status of the deployment in the ArgoCD UI.

## Triggering a New Release

To update your addons or applications, simply commit your changes and create a new Git tag with a `v` prefix:

```bash
# Make your changes and commit them
git add .
git commit -m "feat: update cert-manager version"

# Create and push a new tag
git tag v1.1.0
git push origin v1.1.0
```

The `ApplicationSet` will automatically detect the new `v1.1.0` tag and update the deployed addon applications to this new version.

**Note:** If you change the core setup in the `/argocd` directory, you will also need to manually update the `targetRevision` in `app-of-apps.yaml` and re-apply it.