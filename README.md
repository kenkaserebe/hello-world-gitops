# multi-cloud-gitops-platform/hello-world-gitops/README.md

# Hello World - GitOps Configuration

This repository contains the Kubernetes manifests and ArgoCD `Application` definitions for deploying the [hello-world-app](https://github.com/kenkaserebe/hello-world-app) to both AWS and Azure Kubernetes clusters.

ArgoCD continuously monitors this repository. When the image tag in a `deployment.yaml` is updated (e.g., by the CI pipeline), ArgoCD automatically synchronises the change to the target cluster.


## Repository Structure

hello-world-gitops/
|---aws/                        # AWS EKS deployment
|   |---argocd-app.yaml         # ArgoCD Application definition for AWS
|   |---deployment.yaml         # Kubernetes Deployment (image tag updated by CI)
|   |---service.yaml            # LoadBalancer Service
|
|---azure/                      # Azure AKS deployment
    |---argocd-app.yaml         # ArgoCD Application definition for Azure
    |---deployment.yaml         # Kubernetes Deployment
    |---service.yaml            # LoadBalancer Service

## What Each File Does

### argocd-app.yaml

Defines an ArgoCD `Application` resource. It tell ArgoCD:
    - Where to find the Kubernetes manifests (`source.path` = `aws/` or `azure`).
    - Which cluster to deploy to (`destination.server`).
        In this example, it uses `https://kubernetes.default.svc` - the local cluster where ArgoCD is running.
        **For multi-cluster**, you would replace this with the actual cluster API endpoint or use a cluster secret.
    - Sync policy: `automated` with `prune` and `selfHeal` enabled.


### deployment.yaml

Deploys the hello-world container. The `image` field is the only part updated by the CI pipeline (the tag changes with each git commit SHA).
The deployment runs 2 replicas and exposes port 5000.


### service.yaml

Creates a `LoadBalancer` Service, making the application accessible from outside the cluster on port 80 (mapped to container port 5000).


## Usage

### Prerequisites

- ArgoCD installed on both AWS EKS and Azure AKS clusters (see `argocd/aws` and `argocd/azure` Terraform modules).
- This GitOps repository must be accessible by ArgoCD (public or private with authentication).


### 1. Apply ArgoCD Applications

After ArgoCD is running, apply the `Application` manifests to the respective clusters:

#### On AWS EKS

bash
kubectl apply -f aws/argocd-app.yaml

#### On Azure AKS

bash
kubectl apply -f aws/argocd-app.yaml


ArgoCD will then automatically create the hello-world Deployment and Service on each cluster.


### 2. Automatic Image Updates

The CI pipeline in the hello-world-app repository:
    - Builds a new Docker image.
    - Pushes it to Amazon ECR and Azure ACR.
    - Updates the `image:` field in `aws/deployment.yaml` and `azure/deployment.yaml` with the new tag.
    - Commits and pushes the change to this repository.

Because ArgoCD watches this repo (with automated sync), it will immediately deploy the new image tag to both clusters.


### 3. Manual Image Update

If you want to manually roll back or update the image, edit the `deployment.yaml` file and commit the change:

bash
git clone https://github.com/kenkaserebe/hello-world-gitops.git
cd hello-world-gitops
# Edit aws/deployment.yaml and/or azure/deployment.yaml
git add .
git commit -m "Update image tag to <new-tag>"
git push

ArgoCD will sync the change within a few seconds (depending on the sync interval).


## Accessing the Application

After the Service is provisioned, get the external address:

### AWS (EKS)

bash
kubectl get svc hello-world -n default -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

Then open http://<hostname> in the browser.


### Azure (AKS)

bash
kubectl get svc hello-world -n default -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

Then open http://<ip>.

You should see:

Hello from aws!     (on AWS)
Hello from azure!   (on Azure)


## Customising the ArgoCD Applications

- Target cluster: If ArgoCD manages multiple clusters, replace `server: https://kubernetes.default.svc` with the actual cluster API URL (or reference a cluster secret name).
- Namespace: Change `namespace: default` to any existing namespace (or let `CreateNamespace=true` create it).
- Sync interval: ArgoCD checks for changes every 3 minutes by default. You can adjust this via ArgoCD global settings or add a `syncPolicy.syncOptions` like `PruneLast=true`.


## Security

- The ArgoCD instance must have read access to this repository. If the repo is private, configure a repository credential in ArgoCD (using a GitHub Personal Access Token).
- The `hello-world` Service uses a LoadBalancer, which creates a public IP/hostname. For production, consider using an Ingress Controller with TLS.


## Integration with CI Pipeline

This repository is designed to work with the GitHub Actions workflow defined in the `hello-world-app` repository. The workflow:
- Clones this repo using a GITOPS_REPO_PAT secret.
- Updates the `image:` tags in both `aws/deployment.yaml` and `azure/deployment.yaml` using `yq`.
- Commits and pushes the changes.

Ensure the `GITOPS_REPO_PAT` has `write` permissions to this repository.


## License

[Specify your license, e.g., MIT]