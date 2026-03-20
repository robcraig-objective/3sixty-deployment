# 3Sixty Production Deployment — Quick Start

This guide covers the fastest path to a running 3Sixty stack on Kubernetes (AKS).
Full details are in [README.md](README.md) and [SECRETS-MANAGEMENT.md](SECRETS-MANAGEMENT.md).

---

## Prerequisites

Install required tools (Windows — runs via Winget):

```powershell
make setup-kubectl
```

Or manually install:

- [kubectl](https://kubernetes.io/docs/tasks/tools/) ≥ 1.28
- [Helm](https://helm.sh/docs/intro/install/) ≥ 3.12
- [AWS CLI](https://aws.amazon.com/cli/) ≥ 2 (for ECR image pull credentials)
- [OpenSSL](https://www.openssl.org/) (for TLS certificate generation)
- [PowerShell 7+](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell) (`pwsh`)

---

## Step 1 — Connect to the Cluster

### Check your current cluster context

Before doing anything, confirm you are pointed at the right cluster:

```powershell
# Show the active context (cluster + user + namespace)
kubectl config current-context

# List all available contexts and highlight the active one
kubectl config get-contexts
```

### Switch to a different cluster

```powershell
# Switch to a named context from your kubeconfig
kubectl config use-context <context-name>
```

### Direct kubectl access (local kubeconfig)

```powershell
# Verify cluster connectivity after switching context
kubectl cluster-info
```

### AKS without direct network access

```powershell
# Fetches AKS credentials and sets the active context automatically
make aks-credentials AKS_RG=<resource-group> AKS_CLUSTER=<cluster-name>
```

---

## Step 2 — Create Kubernetes Secrets

All credentials are stored in Kubernetes Secrets created **before** Helm runs.

### Namespace

The secrets scripts **automatically create the namespace** if it does not exist. The default namespace is `threesixty`.

If you need a different namespace, update it in **all three places** before running anything:

| File | Setting |
| ---- | ------- |
| `kubectl-create-secrets.ps1` | `$NAMESPACE = "threesixty"` near the top of the file |
| `aks-create-secrets.ps1` | `-Namespace` parameter (default: `"threesixty"`) |
| Makefile / `make` commands | `NAMESPACE ?= threesixty` — or pass `NAMESPACE=<ns>` to every `make` call |

`values-production.yaml` does not contain a namespace setting — Helm reads it from the `--namespace` flag, which the Makefile supplies via the `NAMESPACE` variable.

### Option A — Direct kubectl access

1. Open `kubectl-create-secrets.ps1` and fill in all credential variables at the top of the file.
2. Run:

   ```powershell
   make secrets-kubectl
   ```

### Option B — AKS invoke (no direct kubectl access)

1. Open `aks-create-secrets.ps1` and fill in all credential variables at the top of the file.
2. Run:

   ```powershell
   make secrets-aks
   ```

### Verify secrets were created

```powershell
make verify-secrets
```

All 7 secrets should show `[OK]`. See [SECRETS-MANAGEMENT.md](SECRETS-MANAGEMENT.md) for the full list.

---

## Step 3 — Configure values-production.yaml

```powershell
# Copy the template
Copy-Item values-production.yaml.template values-production.yaml

# Edit with your values
code values-production.yaml
```

Required values to fill in:

| Section | Required fields |
| ------- | --------------- |
| `traefik` | `ingressClassName: threesixty-stack-traefik` (set automatically). Ports: `exposedPort.http: 7070`, `exposedPort.https: 7443` (change for AKS/EKS LoadBalancer deployments). |
| `threesixty.rabbitmq` | `username`, `password` — **must match** `$RABBITMQ_USER`/`$RABBITMQ_PASS` in setup script |
| `threesixty.oauth2` | `clientId`, `tenantId`, `clientSecret` — from Azure AD App Registration |
| `threesixty.env.admin` | `MONGO_USERNAME`, `MONGO_PASSWORD`, `MONGO_HOST` |
| `hybridsearch.oirag` | `OPENSEARCH_USER` — always `admin` |
| `hybridsearch.remoteagent.enabled` | `false` until a real token is created in the admin UI |

**Do not** commit `values-production.yaml` — it is in `.gitignore`.

---

## Step 4 — Install

```powershell
# Download subchart dependencies (only needed once, or after Chart.yaml changes)
make deps

# Install the Helm release
make install
```

This runs `helm install` with `--wait --timeout 10m`. Pods that take longer to start
(Ollama model pull, OpenSearch JVM startup) are covered by startup probes.

---

## Step 5 — Verify

```powershell
# Check all pods are Running/Ready
make status

# Watch pods in real time
make pods

# Check logs for a specific component
make logs DEPLOY=admin
make logs DEPLOY=oi-rag
make logs DEPLOY=search
```

---

## Accessing the Application

Once all pods are running, access the 3Sixty admin UI via Traefik:

### Add domain to hosts file (local/Docker Desktop deployments)

```powershell
# Windows: C:\Windows\System32\drivers\etc\hosts
127.0.0.1 threesixty.objective.com
```

Or use the `hosts` PowerShell module:

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 threesixty.objective.com"
```

### Connect to the admin UI

- **Local/Docker Desktop**: `https://threesixty.objective.com:7443/3sixty-admin` (port 7443)
- **AKS/EKS with LoadBalancer**: `https://threesixty.objective.com/3sixty-admin` (port 443)

The browser will warn about the self-signed certificate (if used) — click **Advanced** → **Proceed** to continue.

**Default credentials**: Use the OAuth2 user configured in your Azure AD App Registration.

---

## Common Operations

| Task | Command |
| ---- | ------- |
| Deploy config change | `make upgrade` |
| Check pod logs | `make logs DEPLOY=<component>` |
| View Kubernetes events | `make events` |
| Roll back to previous version | `make rollback` |
| View release history | `make history` |
| Dry-run / validate templates | `make validate` |
| Uninstall (keep data) | `make uninstall` |
| Uninstall + delete all data | `make purge` |

Run `make help` for the full list of targets and configuration variables.

---

## Azure AD / Entra ID App Registration

The 3Sixty Discovery service uses OAuth2/OIDC for user authentication. You need an App Registration in your Azure AD tenant.

1. In the Azure Portal, go to **Azure Active Directory → App registrations → New registration**
2. Set the Redirect URI to: `https://<your-domain>/3sixty-discovery/login/oauth2/code/azure`
3. Under **Certificates & secrets**, create a new client secret
4. Copy the **Application (client) ID**, **Directory (tenant) ID**, and the client secret value into `values-production.yaml`

---

## TLS Certificate

### Creating a Self-Signed Certificate on Windows

The `kubectl-create-secrets.ps1` script requires a pre-existing `traefik-tls` Kubernetes secret. Generate the certificate using PowerShell (no OpenSSL needed for initial generation) and optionally convert to PEM format:

```powershell
# Generate self-signed certificate (PowerShell — no external tools needed for this step)
$cert = New-SelfSignedCertificate -DnsName "threesixty.objective.com" -CertStoreLocation "Cert:\CurrentUser\My" -NotAfter (Get-Date).AddYears(2)
$pwd = ConvertTo-SecureString -String "temp1234" -Force -AsPlainText
Export-PfxCertificate -Cert $cert -FilePath "$env:TEMP\traefik.pfx" -Password $pwd

# Convert to PEM format (requires OpenSSL — install via Chocolatey: choco install openssl -y)
openssl pkcs12 -in "$env:TEMP\traefik.pfx" -nokeys -out "$env:TEMP\tls.crt" -passin pass:temp1234
openssl pkcs12 -in "$env:TEMP\traefik.pfx" -nocerts -nodes -out "$env:TEMP\tls.key" -passin pass:temp1234

# Create Kubernetes secret
kubectl create secret tls traefik-tls --cert="$env:TEMP\tls.crt" --key="$env:TEMP\tls.key" -n threesixty
```

### Using a CA-Signed Certificate

For production, replace the self-signed certificate:

```powershell
kubectl create secret tls traefik-tls `
    --cert=/path/to/cert.crt `
    --key=/path/to/cert.key `
    --namespace threesixty `
    --dry-run=client -o yaml | kubectl apply -f -
```

---

## Troubleshooting

**Pod stuck in `Pending`**
Check storage class and PVC binding: `kubectl get pvc -n threesixty`

**Pod stuck in `Init:0/1` or `CrashLoopBackOff`**
Check events and logs: `make events` then `make logs DEPLOY=<name>`

**ECR pull fails (`ImagePullBackOff`)**
The ECR token expires after 12 hours. Re-run the secrets script to refresh it.

**OpenSearch fails to start**
The cluster node needs `vm.max_map_count ≥ 262144`. Set it via a DaemonSet or:
`kubectl exec -n threesixty <node-pod> -- sysctl -w vm.max_map_count=262144`

**Ollama startup takes > 5 minutes**
This is expected on the first start — Ollama pulls the LLM model (~4 GB). The startup probe allows up to 10 minutes. The model is cached in the PVC for subsequent restarts.
