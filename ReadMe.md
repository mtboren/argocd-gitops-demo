# argocd-gitops-demo
Testing out some Argo CD GitOps

Has a [devcontainer.json](.devcontainer/devcontainer.json) that sets up Docker / Kind / kubectl for the exploration

## The Flow
The flow of things from this particular tutorial

### Create the Git Repository

Argo CD requires a Git repository to act as the single source of truth.

- Create a new repository on GitHub named `some-cool-repo`
- Clone it to your local machine (or launch a Codespace), and create a folder named `apps`
- In the `apps` folder, create a file named `web-app.yaml`. This file defines a deployment and a service for an Nginx web server (content below)
- add / commit / push
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-web-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: sample-web-service
  namespace: default
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: web-server
```

### Spin Up a Local Kubernetes Cluster

Spin up a new isolated cluster using `Kind`:
```PowerShell
# Create a local Kubernetes cluster named 'gitops-local'
kind create cluster --name gitops-local

# Verify kubectl is pointing to the new cluster
kubectl cluster-info --context kind-gitops-local
```

### Install Argo CD into the Cluster

Next, install Argo CD into its own dedicated namespace within the new local cluster:
```PowerShell
# Create the argocd namespace
kubectl create namespace argocd

# Apply the official Argo CD installation manifests
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all Argo CD pods to become fully ready and operational
Write-Verbose -Verbose "Waiting for Argo CD components to start..."
kubectl wait --namespace argocd --for=condition=ready pod --all --timeout=120s
```