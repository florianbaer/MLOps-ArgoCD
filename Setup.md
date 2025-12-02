# Setup Docker K8s Cluster

**important**: enable coredns-> `microk8s enable dns`

# Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Set Context

`kubectl config set-context --current --namespace=argocd`

## Install CLI
(CLI Docs)[https://argo-cd.readthedocs.io/en/stable/getting_started/#2-download-argo-cd-cli]

## Port Forwarding

`kubectl port-forward svc/argocd-server -n argocd 8080:443`

## Get initial Password
`argocd admin initial-password -n argocd`

## Login
`argocd login localhost:8080`
and continue with insecure connection (only local!)

## Change PW

`argocd account update-password`

## Add cluster to ArgoCD CLI

`argocd cluster add docker-desktop`

# Deploy Airflow

## Step 1: Update Helm Dependencies (Local Testing Only)

For local testing, you can download the Airflow chart dependency:

```bash
helm dependency update
```

This creates `Chart.lock` (which should be committed) and `charts/` (which is gitignored).


## Step 2: Commit and Push to Git

**Note:** The `charts/` directory is gitignored. ArgoCD will automatically download dependencies when deploying.

```bash
git add Chart.yaml Chart.lock values.yaml argocd-application.yaml .gitignore Setup.md
git commit -m "Update Airflow Helm chart configuration"
git push
```

## Step 3: Deploy with ArgoCD

The SSH secret for DAGs repository is configured in `values.yaml` and will be created automatically by the Helm chart.

```bash
# Apply the ArgoCD Application
kubectl apply -f argocd-application.yaml

# Sync the application (optional - auto-sync is enabled)
argocd app sync mlops-airflow

# Check status
argocd app get mlops-airflow
```



