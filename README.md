# MLOps Airflow with Flux CD

GitOps-based deployment of Apache Airflow for MLOps workflows, running locally on a Podman + kind Kubernetes cluster.

## What you'll build

A working Apache Airflow instance on your laptop:

- **Kubernetes cluster** running locally in a single Podman container (via `kind`)
- **Flux CD** managing the Airflow deployment
- **Apache Airflow** (Helm chart) with PostgreSQL, scheduler, DAG processor, API server
- **Git-sync** pulling DAG files from your GitHub repository over SSH

Expected total time: **15–25 minutes** (most of it waiting for images to pull).

---

## Prerequisites

You need a Mac (these instructions assume macOS with Homebrew). Linux works too but command names may differ.

Install these once:

```bash
brew install podman kind kubectl fluxcd/tap/flux
```

Verify:

```bash
podman --version
kind --version
kubectl version --client
flux --version
```

You also need a **GitHub account** and a **repository** that will hold your DAG Python files (see Step 4).

---

## Step 1 — Start Podman with enough resources

Airflow + Postgres + the Kubernetes control plane need roughly 8 GB of memory. The default Podman machine is too small.

```bash
podman machine init         # only if you've never used podman before
podman machine stop         # ignore error if not yet running
podman machine set --memory 12288 --cpus 6
podman machine start
```

Verify:

```bash
podman machine inspect podman-machine-default | grep -E "Memory|CPUs"
# expected: "CPUs": 6,  "Memory": 12288,
```

---

## Step 2 — Create a local Kubernetes cluster with kind

```bash
KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster \
  --image kindest/node:v1.30.13 --name mlops-flux
```

This takes 2–3 minutes on first run (downloads the node image). When done:

```bash
kubectl cluster-info
# Kubernetes control plane is running at https://127.0.0.1:<port>
```

> **If your laptop sleeps or you reboot**, the kind container may stop.
> Restart it with:
> ```bash
> podman start mlops-flux-control-plane
> ```

---

## Step 3 — Install Flux into the cluster

We're **not** using `flux bootstrap` — that would push a `flux-system/` folder to your GitHub repo. For a local student setup we just install Flux directly:

```bash
flux check --pre          # should all be green
flux install
```

Expected final lines:

```
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ install finished
```

---

## Step 4 — Prepare your DAG repository

Create a GitHub repository that will hold your Airflow DAG Python files. For example: `github.com/<your-username>/MLOps-Dags`.

Add at least one DAG file so Airflow has something to parse. A minimal example:

```python
# hello_dag.py
from datetime import datetime
from airflow import DAG
from airflow.operators.empty import EmptyOperator

with DAG("hello", start_date=datetime(2024, 1, 1), schedule=None, catchup=False):
    EmptyOperator(task_id="noop")
```

Commit and push it to `main`.

### 4.1 — Update the DAG repo URL in this project

Edit `clusters/production/03-helmrelease.yaml` and change the `repo:` line under `dags.gitSync` to point to **your** repo:

```yaml
dags:
  gitSync:
    enabled: true
    repo: git@github.com:<your-username>/<your-dag-repo>.git
    branch: main
    sshKeySecret: flux-git-ssh
```

### 4.2 — Generate an SSH key for git-sync

```bash
ssh-keygen -t ed25519 -C "airflow-dag-sync" -f ~/.ssh/airflow_dag_key -N ""
cat ~/.ssh/airflow_dag_key.pub
```

### 4.3 — Add the public key as a Deploy Key on your DAG repo

A **Deploy Key** is an SSH public key attached to a single repository. It lets the Airflow pod read DAGs from that repo without needing your personal GitHub credentials.

1. Go to `https://github.com/<your-username>/<your-dag-repo>/settings/keys`
2. Click **"Add deploy key"**
3. **Title**: `Airflow GitSync`
4. **Key**: paste the output of `cat ~/.ssh/airflow_dag_key.pub`
5. **Leave "Allow write access" UNCHECKED** (read-only is safer)
6. Click **"Add key"**

### 4.4 — Create the SSH secret file

```bash
cp secret.yaml.template secret.yaml
```

Open `secret.yaml` in an editor, uncomment the lines, and paste your **private** key under `gitSshKey: |` (keep the 4-space indent):

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
    [paste contents of ~/.ssh/airflow_dag_key here]
    -----END OPENSSH PRIVATE KEY-----
```

> **IMPORTANT:** `secret.yaml` is already in `.gitignore`. **Never commit your private key.**
>
> Double-check:
> ```bash
> git status   # secret.yaml must NOT appear as untracked or modified
> ```

---

## Step 5 — Create Kubernetes namespace and secrets

Airflow needs a namespace and four secrets. The namespace is created first; then all secrets go into it.

```bash
kubectl create namespace airflow

kubectl apply -f secret.yaml

kubectl create secret generic airflow-postgresql \
  --from-literal=postgres-password="$(openssl rand -base64 32)" \
  --from-literal=password="$(openssl rand -base64 32)" \
  --namespace airflow

kubectl create secret generic airflow-webserver-secret \
  --from-literal=webserver-secret-key="$(python3 -c 'import secrets; print(secrets.token_hex(32))')" \
  --namespace airflow

kubectl create secret generic airflow-fernet-key \
  --from-literal=fernet-key="$(python3 -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())')" \
  --namespace airflow
```

Verify all four secrets exist:

```bash
kubectl get secrets -n airflow
```

Expected:

```
NAME                       TYPE     DATA   AGE
airflow-fernet-key         Opaque   1      5s
airflow-postgresql         Opaque   2      10s
airflow-webserver-secret   Opaque   1      7s
flux-git-ssh               Opaque   1      15s
```

---

## Step 6 — Deploy Airflow via Flux

Apply the HelmRepository (where to fetch the chart from) and the HelmRelease (the Airflow deployment itself):

```bash
kubectl apply -f clusters/production/01-helmrepository.yaml
kubectl apply -f clusters/production/03-helmrelease.yaml
```

> The `02-gitrepository.yaml` file is **optional** and only needed for `flux bootstrap`. Skip it for local dev.

Watch Flux do its work:

```bash
flux get helmreleases -A
```

First you'll see `Running 'install' action` for up to several minutes while images pull and pods start.

```bash
kubectl get pods -n airflow
```

You're aiming for this state (5 `Running`, 1 `Completed`):

```
NAME                                         READY   STATUS     RESTARTS   AGE
mlops-airflow-api-server-xxx                 1/1     Running    0          5m
mlops-airflow-create-user-xxx                0/1     Completed  1          5m
mlops-airflow-dag-processor-xxx              3/3     Running    0          5m
mlops-airflow-postgresql-0                   1/1     Running    0          5m
mlops-airflow-run-airflow-migrations-xxx     0/1     Completed  0          5m
mlops-airflow-scheduler-xxx                  2/2     Running    0          5m
mlops-airflow-statsd-xxx                     1/1     Running    0          5m
mlops-airflow-triggerer-0                    3/3     Running    0          5m
```

This takes **5–10 minutes** on first deploy. Pulling the Airflow image (`~520 MB`) and Postgres image is the slow part.

---

## Step 7 — Open the Airflow UI

```bash
kubectl port-forward -n airflow svc/mlops-airflow-api-server 8080:8080
```

Open in your browser: **http://localhost:8080**

Default credentials:
- **Username:** `admin`
- **Password:** `admin`

Your DAGs from the GitHub repo should appear on the dashboard within 30–60 seconds.

---

## Step 8 — Verify DAG sync

Check the git-sync sidecar inside the DAG processor pod:

```bash
DP=$(kubectl get pods -n airflow -l component=dag-processor -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n airflow $DP -c git-sync --tail=20
```

Look for:

```
"msg":"updated successfully","ref":"main","remote":"<commit-sha>","syncCount":1
```

If you push a new commit to your DAG repo, git-sync will pull it within 60 seconds. No redeploy needed.

---

## Common issues

### 1. Pods stuck in `Pending` with PVC provisioning error

```
failed to provision volume with StorageClass "standard":
NodePath only supports ReadWriteOnce and ReadWriteOncePod access modes
```

**Cause:** kind's `local-path` provisioner does not support `ReadWriteMany`. Airflow's chart tries to create a shared logs PVC with RWX.

**Fix:** This project already sets `logs.persistence.enabled: false` in `03-helmrelease.yaml`. If you see this error, make sure that line is present.

### 2. Helm install fails: `wrong type for value; expected string; got bool`

The Airflow chart expects every value under `config:` to be a **string**. Use `"True"` / `"False"` / `"9125"`, not unquoted `true` / `false` / `9125`. See the `config:` block in `03-helmrelease.yaml` for the correct syntax.

### 3. Kind container exited (OOMKilled, exit code 137)

Your Podman machine ran out of memory. Re-run Step 1 with a larger `--memory` value.

### 4. `The connection to the server 127.0.0.1:<port> was refused`

The kind container stopped (laptop slept, reboot, etc.). Restart:

```bash
podman start mlops-flux-control-plane
```

### 5. DAG parse error: `use_uv=True` unknown

The `@task.virtualenv(use_uv=True)` kwarg requires `apache-airflow-providers-standard >= 1.3` (ships with Airflow 3.1+). The default chart deploys **Airflow 3.0.2**, which has provider 1.2. Either:

- Remove `use_uv=True` from your DAGs, **or**
- Add `defaultAirflowTag: "3.1.0"` under `values:` in `03-helmrelease.yaml` to use a newer Airflow image.

### 6. Stuck HelmRelease after a failed install

If a HelmRelease gets stuck in `Running 'install' action` forever:

```bash
kubectl delete helmrelease mlops-airflow -n airflow
kubectl delete secret -n airflow -l owner=helm
kubectl delete deployment,statefulset,service -n airflow -l release=mlops-airflow
kubectl apply -f clusters/production/03-helmrelease.yaml
```

Your user secrets (`airflow-postgresql`, `airflow-fernet-key`, etc.) stay intact.

---

## Cleanup

Delete everything when you're done:

```bash
KIND_EXPERIMENTAL_PROVIDER=podman kind delete cluster --name mlops-flux
```

To also shut down Podman:

```bash
podman machine stop
```

---

## Repository structure

```
.
├── clusters/
│   └── production/
│       ├── 00-namespace.yaml        # airflow namespace
│       ├── 01-helmrepository.yaml   # Airflow Helm chart source
│       ├── 02-gitrepository.yaml    # (optional, for flux bootstrap)
│       ├── 03-helmrelease.yaml      # Airflow deployment config — edit DAG repo URL here
│       └── flux-deploy-key.yaml     # (optional template)
├── secret.yaml.template             # template for your git-sync SSH secret
└── README.md
```

---

## Going further (optional)

Once your local setup works, you can try:

- **GitOps bootstrap**: use `flux bootstrap github` to have Flux pull the manifests from your fork of this repo instead of applying them with `kubectl apply`.
- **Change Airflow version**: edit `defaultAirflowTag` in `03-helmrelease.yaml`, push, and watch Flux roll out the new version.
- **Add resource limits**: tune the `resources:` blocks in `03-helmrelease.yaml` for your laptop.

## References

- [Flux Documentation](https://fluxcd.io/flux/)
- [Airflow Helm Chart](https://airflow.apache.org/docs/helm-chart/)
- [kind with Podman](https://kind.sigs.k8s.io/docs/user/rootless/#creating-a-kind-cluster-with-rootless-podman)
- [GitHub Deploy Keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
