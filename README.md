# gitops-accounting-app

Kubernetes manifests for the Accounting app, deployed to EKS with Argo CD (GitOps).

## Files
- `deployment.yaml` - runs 2 replicas of the app image from Docker Hub
- `service.yaml` - exposes the app through an AWS load balancer (port 80 -> 8080)

## Flow
1. Jenkins CI builds the app, pushes the Docker image and updates the image tag in `deployment.yaml`.
2. Argo CD watches this repo and syncs any change to the EKS cluster.
