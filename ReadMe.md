# argocd-gitops-demo
Testing out some Argo CD GitOps

Has a [devcontainer.json](.devcontainer/devcontainer.json) that sets up Docker / Kind / kubectl for the exploration

## The Flow
The flow of things from this particular tutorial
- [Create the Git Repository](#create-the-git-repository)
- [Spin Up a Local Kubernetes Cluster](#spin-up-a-local-kubernetes-cluster)
- [Install Argo CD into the Cluster](#install-argo-cd-into-the-cluster)
- [Access the Argo CD Dashboard](#access-the-argo-cd-dashboard)
- [Define the Argo CD Root Application](#define-the-argo-cd-root-application)
  - [Create Argo CD Application spec](#create-argo-cd-application-spec)
  - [Apply this configuration to the cluster](#apply-this-configuration-to-the-cluster)
- [Test Continuous Reconciliation & Self-Healing](#test-continuous-reconciliation--self-healing)



## Create the Git Repository

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

## Spin Up a Local Kubernetes Cluster

Spin up a new isolated cluster using `Kind`:
```PowerShell
# Create a local Kubernetes cluster named 'gitops-local'
kind create cluster --name gitops-local

# Verify kubectl is pointing to the new cluster
kubectl cluster-info --context kind-gitops-local
```

## Install Argo CD into the Cluster

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

## Access the Argo CD Dashboard
By default, the Argo CD API server is not exposed with an external IP. We will use port forwarding to access the user interface.

1. Retrieve the auto-generated admin password
    
    Argo CD creates an initial admin password stored in a Kubernetes secret. PowerShell snippet to extract and decode it:

    ``` PowerShell
    [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
    ```

1. Start Port Forwarding:

    ```PowerShell
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ## Keep this terminal window open to maintain the connection
    ```

1. Log In
    
    Open your browser and navigate to https://localhost:8080 (or, if using a GitHub Codespace, goto the URL for the forwarded port 8080, available from "Ports" tab). Bypass the SSL certificate warning (expected for local development). Log in with the username admin and the password decoded above

## Define the Argo CD Root Application

We must tell Argo CD where to look in Git, and where to deploy those resources in the cluster. We do this by creating an Argo CD Application custom resource.

### Create Argo CD Application spec
Create a file named `argo-application.yaml` (replace `GITHUB_USERNAME` with your actual GitHub username/org):
```yaml
YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: local-web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/GITHUB_USERNAME/argocd-gitops-demo.git'
    targetRevision: HEAD
    path: apps
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

> [!NOTE]
> Key Architecture Note: Notice selfHeal: true. This setting commands Argo CD to actively watch for variances between the cluster and Git, and automatically overwrite any unauthorized cluster changes.

### Apply this configuration to the cluster
Now apply this config:
```PowerShell
kubectl apply -f argo-application.yaml
```
In your browser dashboard, you will instantly see `local-web-app` appear. Within seconds, it will synchronize, pulling the manifests from GitHub and spinning up your two Nginx web server pods.

## Test Continuous Reconciliation & Self-Healing
To see the platform engine in action, we will intentionally bypass Git and alter the infrastructure directly using the Kubernetes API—simulating a developer manually messing with resources.

Let's manually scale our deployment down from 2 replicas to 0 replicas using the CLI:

```PowerShell
# Intentionally cause configuration drift
kubectl scale deployment sample-web-app --replicas=0
```
The Result:
1. Run `kubectl get pods` immediately afterward
1. You will observe that instead of dropping to zero, the pods are either still running or instantly terminating and recreating
1. Check the Argo CD browser dashboard. You will see the application briefly flash a yellow `OutOfSync` status before changing right back to a green `Synced` status.

Argo CD caught the live cluster deviation, consulted your GitHub repository (`apps/web-app.yaml` which mandates `replicas: 2`), determined the cluster was in an illegal state, and executed an automatic remediation loop to scale the pods back up instantly.

## Success
Voila! You've experienced some CI/CD with Argo! You can kill the Codespace, now