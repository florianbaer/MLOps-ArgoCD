# MLOps Airflow with Flux CD

GitOps-based deployment of Apache Airflow for MLOps workflows using Flux CD.

## Overview

This repository contains Flux CD manifests to deploy Apache Airflow on Kubernetes with automated DAG synchronization from a Git repository.

### Architecture

- **GitOps Tool**: Flux CD
- **Workload**: Apache Airflow (Helm Chart)
- **DAG Source**: Git repository with SSH authentication
- **Namespace**: `airflow`

## Prerequisites

### 1. Kubernetes Cluster

**For Docker Desktop (Recommended for Mac/Windows):**
1. Open Docker Desktop settings
2. Go to Kubernetes tab
3. Enable Kubernetes
4. Click "Apply & Restart"
5. Verify: `kubectl cluster-info`

**For kind (Local Development):**
```bash
# Install kind
brew install kind  # macOS

# Create cluster with Kubernetes 1.30+
kind create cluster --image kindest/node:v1.30.13
kubectl cluster-info --context kind-kind
```

**For MicroK8s (Linux):**
```bash
# Enable required addons
microk8s enable dns
microk8s enable storage

# Get kubectl config
microk8s kubectl config view --raw > ~/.kube/config
```

### 2. Flux CLI

**macOS:**
```bash
brew install fluxcd/tap/flux
```

**Linux:**
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Verify installation:**
```bash
flux --version
```

### 3. GitHub Personal Access Token

Create a GitHub Personal Access Token with `repo` permissions:

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Select scopes: `repo` (all)
4. Generate and copy the token

## Installation

### Step 1: Bootstrap Flux

Set up Flux on your cluster and configure it to watch this repository:

```bash
# Set environment variables
export GITHUB_TOKEN=<your-personal-access-token>
export GITHUB_USER=florianbaer

# Verify prerequisites
flux check --pre

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=MLOps-Flux \
  --branch=main \
  --path=./clusters/production \
  --personal
```

**What happens during bootstrap:**
- Flux components are installed in the `flux-system` namespace
- A deploy key is added to your GitHub repository
- Flux starts monitoring the `./clusters/production` directory
- Automatic synchronization is enabled

### Step 2: Apply the SSH Secret for DAG Sync

The Airflow deployment needs SSH access to pull DAGs from a private Git repository.

```bash
# Apply the SSH secret
kubectl apply -f git-ssh-secret.yaml
```

**Verify the secret:**
```bash
kubectl get secret flux-git-ssh -n airflow
```

### Step 3: Create Required Secrets

Before Airflow can deploy, you must create Kubernetes secrets for sensitive data. **These secrets are never stored in Git.**

```bash
# 1. PostgreSQL Database Password
kubectl create secret generic airflow-postgresql \
  --from-literal=postgres-password="$(openssl rand -base64 32)" \
  --from-literal=password="$(openssl rand -base64 32)" \
  --namespace airflow

# 2. Webserver Secret Key (for Flask sessions)
kubectl create secret generic airflow-webserver-secret \
  --from-literal=webserver-secret-key="$(python3 -c 'import secrets; print(secrets.token_hex(32))')" \
  --namespace airflow

# 3. Fernet Key (for encrypting connections/variables in Airflow)
kubectl create secret generic airflow-fernet-key \
  --from-literal=fernet-key="$(python3 -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())')" \
  --namespace airflow
```

**Verify all secrets are created:**
```bash
kubectl get secrets -n airflow

# Expected output should include:
# - airflow-postgresql
# - airflow-webserver-secret
# - airflow-fernet-key
# - flux-git-ssh
```

**IMPORTANT:** Save the generated passwords securely (use a password manager). You'll need them if you ever need to access the database directly or restore the secrets.

### Step 4: Wait for Flux to Deploy Airflow

Flux will automatically detect the manifests in `clusters/production/` and deploy them:

```bash
# Watch Flux reconciliation
flux logs --follow

# Check Helm releases
flux get helmreleases -n airflow

# Check all Flux sources
flux get sources all
```

### Step 5: Verify Airflow Deployment

```bash
# Check pods in airflow namespace
kubectl get pods -n airflow

# Check HelmRelease status
kubectl describe helmrelease mlops-airflow -n airflow

# Check git-sync logs (DAG synchronization)
kubectl logs -n airflow <scheduler-pod-name> -c git-sync
```

## Repository Structure

```
.
├── clusters/
│   └── production/
│       ├── 00-namespace.yaml         # Airflow namespace (applied first)
│       ├── 01-helmrepository.yaml    # Airflow Helm repository source
│       ├── 02-gitrepository.yaml     # Config repository source (optional)
│       └── 03-helmrelease.yaml       # Airflow deployment with security
git-ssh-secret.yaml               # SSH key for DAG repository
Chart.yaml                        # Helm chart metadata (optional)
└── README.md                         # This file
```

**Note:** Files in `clusters/production/` are numbered to ensure proper application order:
1. Namespace created first
2. Helm and Git repositories configured
3. HelmRelease deployed last (after dependencies exist)

## Configuration

### Airflow Configuration

Main configuration is in `clusters/production/03-helmrelease.yaml`:

```yaml
spec:
  chart:
    spec:
      chart: airflow
      version: "1.18.0"
  values:
    airflowVersion: "3.0.2"
    executor: "KubernetesExecutor"

    # Security - references to secrets (created separately)
    webserverSecretKeySecretName: "airflow-webserver-secret"
    fernetKeySecretName: "airflow-fernet-key"

    # DAG synchronization
    dags:
      gitSync:
        enabled: true
        repo: git@github.com:HSLU-DBIZ/JobMonitor-DAGs.git
        branch: main
        sshKeySecret: flux-git-ssh

    # Database (uses external secret)
    postgresql:
      auth:
        existingSecret: "airflow-postgresql"
```

**Key features:**
- Airflow 3.0.2 with Helm chart 1.18.0
- KubernetesExecutor for scalable task execution
- Secure secret management (no passwords in Git)
- Git-sync for automatic DAG updates
- PostgreSQL with persistent storage
- Resource limits and health probes configured

### Modifying Configuration

1. Edit `clusters/production/03-helmrelease.yaml`
2. Commit and push changes to GitHub
3. Flux automatically detects changes and reconciles within the interval period (default: 10 minutes)

**Force immediate reconciliation:**
```bash
flux reconcile source git flux-system
flux reconcile helmrelease mlops-airflow -n airflow
```

## Monitoring

### Check Flux Status

```bash
# Overall Flux health
flux check

# Git repository sync status
flux get sources git -n flux-system

# Helm repository sync status
flux get sources helm -n flux-system

# HelmRelease status
flux get helmreleases -n airflow

# Continuous logs
flux logs --follow
```

### Check Airflow Status

```bash
# All resources in airflow namespace
kubectl get all -n airflow

# Specific pods
kubectl get pods -n airflow -w

# Scheduler logs
kubectl logs -n airflow <scheduler-pod> -f

# Git-sync status (DAG synchronization)
kubectl logs -n airflow <scheduler-pod> -c git-sync -f
```

### Access Airflow Web UI

```bash
# Port-forward to access locally
kubectl port-forward -n airflow svc/mlops-airflow-web 8080:8080

# Access at: http://localhost:8080
```

## Security

### Secret Management

All sensitive data (passwords, keys) are stored in Kubernetes secrets and **never committed to Git**. The HelmRelease references these secrets by name.

**Required secrets:**
- `airflow-postgresql` - Database passwords
- `airflow-webserver-secret` - Flask session secret
- `airflow-fernet-key` - Airflow encryption key
- `flux-git-ssh` - SSH key for DAG repository

### Security Best Practices

1. **Use external secrets management** (HashiCorp Vault, AWS Secrets Manager, etc.) for production
2. **Enable secret encryption at rest** in Kubernetes
3. **Rotate secrets regularly** (e.g., every 90 days)
4. **Use RBAC** to limit secret access
5. **Monitor secret access** via audit logs
6. **Consider External Secrets Operator** for automated secret management from external sources

### Production Security Checklist

Before deploying to production:

- [ ] Replace default passwords with strong, randomly generated ones
- [ ] Enable TLS/SSL for all connections
- [ ] Configure network policies to restrict pod communication
- [ ] Set up RBAC with least-privilege access
- [ ] Enable pod security policies/standards
- [ ] Configure resource limits and quotas
- [ ] Set up backup and disaster recovery for PostgreSQL data
- [ ] Enable audit logging

## Updating Airflow Version

Edit `clusters/production/03-helmrelease.yaml`:

```yaml
spec:
  chart:
    spec:
      version: "1.16.0"  # Update version here
```

Commit and push. Flux will automatically upgrade Airflow.

## Uninstalling

### Remove Airflow

```bash
flux delete helmrelease mlops-airflow -n airflow
```

### Uninstall Flux

```bash
flux uninstall --silent
```

## Rollback to ArgoCD

If needed, you can rollback to ArgoCD:

1. Uninstall Flux:
   ```bash
   flux uninstall --silent
   ```

2. Restore ArgoCD application:
   ```bash
   kubectl apply -f archive/argocd-application.yaml
   ```

3. Revert secret changes:
   ```bash
   git checkout archive/git-ssh-secret.yaml
   kubectl apply -f archive/git-ssh-secret.yaml
   ```

## Tips

### Development Workflow

1. **Use shorter intervals for faster development:**
   Edit `clusters/production/02-gitrepository.yaml` and set `interval: 1m` for quicker config updates.

2. **Watch reconciliation in real-time:**
   ```bash
   watch flux get helmreleases -A
   ```

3. **Suspend reconciliation for manual changes:**
   ```bash
   # Suspend to make manual changes
   flux suspend helmrelease mlops-airflow -n airflow

   # Make your changes...

   # Resume when done
   flux resume helmrelease mlops-airflow -n airflow
   ```

4. **Export Helm values for debugging:**
   ```bash
   helm get values mlops-airflow -n airflow
   ```

5. **Check Flux version:**
   ```bash
   flux version
   ```

6. **View scheduler logs with DAG sync info:**
   ```bash
   SCHEDULER_POD=$(kubectl get pods -n airflow -l component=scheduler -o jsonpath='{.items[0].metadata.name}')
   kubectl logs -n airflow $SCHEDULER_POD -c git-sync -f
   ```

## Additional Resources

- [Flux Documentation](https://fluxcd.io/flux/)
- [Flux Get Started Guide](https://fluxcd.io/flux/get-started/)
- [Airflow Helm Chart](https://airflow.apache.org/docs/helm-chart/)
- [Flux Helm Guide](https://fluxcd.io/flux/guides/helmreleases/)
- [GitOps Principles](https://opengitops.dev/)

## Support

For issues related to:
- **Flux**: [Flux GitHub Issues](https://github.com/fluxcd/flux2/issues)
- **Airflow Helm Chart**: [Airflow Helm Chart Issues](https://github.com/apache/airflow/issues)
- **This Repository**: [Open an issue](https://github.com/florianbaer/MLOps-Flux/issues)