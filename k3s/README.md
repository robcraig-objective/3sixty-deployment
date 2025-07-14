# 3Sixty k3s Local Development

This repository contains Kubernetes manifests and instructions to spin up the full 3Sixty stack in a k3s (e.g. Rancher Desktop, Docker Desktop, minikube) cluster for local development and testing.

## Prerequisites
* k3s (e.g. via Rancher Desktop)
* kubectl ≥ 1.24, configured to your k3s context
* AWS CLI (for ECR image pulls)
* OpenSSL (for self-signed certificates)
* (Optional) ngrok or cloudflared (for public URLs)


## 1. TLS Certificates
To terminate HTTPS in our nginx-proxy, you need a certificate + key:
```bash
mkdir -p certs
openssl req -x509 -newkey rsa:2048 -days 365 \
  -nodes \
  -keyout certs/tls.key \
  -out certs/tls.crt \
  -subj "/CN=localhost"
```
Create the TLS secret in Kubernetes:
```bash
kubectl create secret tls nginx-tls \
  --cert=certs/tls.crt \
  --key=certs/tls.key
```

## 2. Private Registry Credentials
### AWS ECR
Generate an ECR token and store it as a pull-secret:
```bash
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=782396859527.dkr.ecr.ap-southeast-2.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region ap-southeast-2)" \
  --namespace default
```
> Tip: ECR tokens expire every 12 hours. Script this command in CI or a cronjob to refresh automatically.

## 3. ConfigMaps & Secrets
Most non-sensitive settings live in ConfigMaps; secrets (OAuth credentials, DB passwords) live in Kubernetes Secrets. You create them individually, for example:
```bash
kubectl create secret generic discovery-secret \
--from-literal=CLIENT_ID="<your-client-id>" \
--from-literal=TENANT_ID="<your-tenant-id>" \
--from-literal=CLIENT_SECRET="<your-client-secret>" \
--namespace default
```

Alternatively configs can be read from environment variables:
```bash
kubectl create configmap oi-rag-config --from-env-file=.env.oirag
...
```
> Note: All .env.* files in the Docker-Compose setup map to these ConfigMaps/Secrets.

## 4. Deploy the Stack
Apply all k3s manifests:
```bash
kubectl apply -f elasticsearch.yaml
kubectl apply -f mongo.yaml
kubectl apply -f opensearch.yaml
kubectl apply -f oi-rag-ollama.yaml
kubectl apply -f threesixty.yaml
kubectl apply -f threesixty-ingress.yaml
```
Watch for readiness:
```bash
kubectl get pods -w
```

## 4. Stopping the Stack
To tear down application components without removing persistent storage (PVCs), run:
```bash
kubectl delete \
-f elasticsearch.yaml \
-f mongo.yaml \
-f opensearch.yaml \
-f oi-rag-ollama.yaml \
-f threesixty.yaml \
-f threesixty-ingress.yaml
```

This will delete Deployments, Services, and related resources defined in those manifests, but leave any PVCs (e.g., your MongoDB data) intact.

Alternatively, you can delete all application resources in the namespace except PVCs with:

```bash
kubectl delete deployments,services,ingress,configmaps,secrets --all
```

If you ever need to remove the PVCs as well, you can explicitly run:
```bash
kubectl delete pvc --all
```

## 5. Troubleshooting
Docker Desktop’s built-in Kubernetes does not include an Ingress controller by default. To enable host- and path-based routing on ports 80/443, install Traefik or another Ingress Controller:

If you don’t have Helm installed, you can install it easily:
* On macOS with Homebrew: `brew install helm`
* Or follow official instructions: https://helm.sh/docs/intro/install/

### Add the Traefik Helm repo
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```
### Install Traefik with Helm
Here’s a basic install into the traefik namespace:
```bash
kubectl create namespace traefik
helm install traefik traefik/traefik --namespace=traefik
```
This deploys Traefik as a Kubernetes deployment + service with default settings.

### Check Traefik pods and service
```bash
kubectl get pods -n traefik
kubectl get svc -n traefik
```
By default, Traefik exposes:
* ports 80 and 443 via a LoadBalancer service if your cluster supports it
* on Docker Desktop, you’ll typically get a ClusterIP service, so use kubectl port-forward or change the service to NodePort to access externally.

### Use Traefik IngressClass
When you create Ingress resources, specify:
```yaml
spec:
  ingressClassName: traefik
```
