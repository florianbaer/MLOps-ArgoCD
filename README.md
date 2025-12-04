# MLOps Airflow with Flux CD

GitOps-based deployment of Apache Airflow for MLOps workflows using Flux CD.

## Overview

This repository contains Flux CD manifests to deploy Apache Airflow on Kubernetes with automated DAG synchronization from a Git repository.

### Architecture

- **GitOps Tool**: Flux CD
- **Workloads**:
  - Apache Airflow (Helm Chart) - MLOps workflows
- **DAG Source**: Git repository with SSH authentication
- **Namespaces**: `airflow`

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
export GITHUB_USER=<your-github-username>

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

### Step 2: Generate SSH Key for DAG Repository

Before applying the SSH secret, you need to generate an SSH key pair and configure it as a GitHub deploy key for your DAG repository.

#### 2.1 Generate SSH Key Pair

```bash
# Generate SSH key (press Enter when prompted for passphrase to leave it empty)
ssh-keygen -t ed25519 -C "airflow-dag-sync" -f ~/.ssh/airflow_dag_key
```

This creates two files:
- `~/.ssh/airflow_dag_key` - Private key (keep secret)
- `~/.ssh/airflow_dag_key.pub` - Public key (add to GitHub)

#### 2.2 Add Deploy Key to GitHub

1. Copy your public key:
   ```bash
   cat ~/.ssh/airflow_dag_key.pub
   ```

2. Add it to your DAG repository on GitHub:
   - Go to your DAG repository → **Settings** → **Deploy keys**
   - Click **"Add deploy key"**
   - **Title**: `Airflow GitSync`
   - **Key**: Paste the public key content
   - **Important**: Leave **"Allow write access"** UNCHECKED (read-only is sufficient)
   - Click **"Add key"**

For detailed instructions, see: [GitHub Deploy Keys Documentation](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)

#### 2.3 Create the Secret File

**IMPORTANT**: This file contains sensitive data and must NEVER be committed to git.

```bash
# Copy the template to create your secret file
cp secret.yaml.template secret.yaml

# Verify it's ignored by git
git status  # secret.yaml should NOT appear in untracked files

# Display your private key
cat ~/.ssh/airflow_dag_key
```

Edit `secret.yaml` and paste the entire private key output (including the BEGIN and END lines) under `gitSshKey`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: flux-git-ssh
  namespace: airflow
type: Opaque
stringData:
  gitSshKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    [paste your private key here - all lines]
    -----END OPENSSH PRIVATE KEY-----
```

**Security Check - Run before committing anything:**
```bash
git status  # Verify secret.yaml is NOT listed
cat .gitignore | grep secret  # Verify secret*.yaml is in .gitignore
```

⚠️ **NEVER commit secret.yaml to git. It contains your private SSH key.**

### Step 3: Apply the SSH Secret for DAG Sync

The Airflow deployment needs SSH access to pull DAGs from a private Git repository.

```bash
# Apply the SSH secret
kubectl apply -f secret.yaml
```

**Verify the secret:**
```bash
kubectl get secret flux-git-ssh -n airflow
```

### Step 4: Create Required Secrets

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

### Step 5: Wait for Flux to Deploy Airflow

Flux will automatically detect the manifests in `clusters/production/` and deploy them.

**Expected Timeline:**
- Initial deployment: 5-10 minutes
- PostgreSQL startup: 2-3 minutes
- Airflow webserver ready: 3-5 minutes after PostgreSQL
- Be patient and watch the logs - multiple pod restarts during initial setup are normal

```bash
# Watch Flux reconciliation
flux logs --follow

# Check Helm releases
flux get helmreleases -n airflow

# Check all Flux sources
flux get sources all
```

### Step 6: Verify Airflow Deployment

```bash
# Check pods in airflow namespace
kubectl get pods -n airflow

# Check HelmRelease status
kubectl describe helmrelease mlops-airflow -n airflow

# Check git-sync logs (DAG synchronization)
kubectl logs -n airflow <scheduler-pod-name> -c git-sync
```

### Step 7: Verify DAGs are Loaded

Once Airflow is running, verify that DAGs are syncing from your Git repository:

1. **Access the Airflow UI** (see "Access Airflow Web UI" section below)

2. **Check for DAGs on the dashboard:**
   - You should see DAGs from your repository listed on the main page
   - Initial sync can take 1-2 minutes after the scheduler starts

3. **Check git-sync logs to confirm synchronization:**
   ```bash
   SCHEDULER_POD=$(kubectl get pods -n airflow -l component=scheduler -o jsonpath='{.items[0].metadata.name}')
   kubectl logs -n airflow $SCHEDULER_POD -c git-sync --tail=50
   ```

4. **Look for log messages indicating successful sync:**
   ```
   INFO: synced 3 files from origin/main
   ```

**Troubleshooting:** If DAGs don't appear:
- Verify SSH key is correctly configured as a deploy key in GitHub
- Check that the repository URL is correct in `clusters/production/03-helmrelease.yaml`
- Review git-sync container logs for authentication errors
- Ensure your DAG repository contains valid Python files with DAG definitions

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

### DAG Repository Setup

This setup syncs DAGs from a Git repository. For this exercise, you can use an example DAG repository or create your own.

**Example DAG Repositories:**
- https://github.com/matsudan/airflow-dag-examples - Collection of example Airflow DAGs
- Fork the above repository to experiment with your own DAG modifications

**To configure your DAG repository:**

1. Choose or create a repository containing Airflow DAG files
2. Update the repository URL in `clusters/production/03-helmrelease.yaml` (line 56):
   ```yaml
   dags:
     gitSync:
       enabled: true
       repo: git@github.com:<your-username>/<your-dag-repo>.git
       branch: main
   ```
3. Follow the SSH key setup in Step 2 to grant read access to the repository

**Note**: The repository should contain Python files with Airflow DAG definitions in the root or in subdirectories.

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

**Default Credentials:**
- **Username**: `admin`
- **Password**: `admin`

**Note**: For production deployments, change the default password by configuring Airflow's RBAC settings and creating custom user accounts.

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
