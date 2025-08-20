# 3Sixty Kubernetes Deployment

This repository contains Helm charts for deploying all 3Sixty services to Kubernetes clusters.

---

## Prerequisites

- **Kubernetes Cluster** (1.24+)
- **Helm** ≥ 3.12
- **kubectl** configured to access your cluster
- **AWS CLI** ≥ 2 (for ECR authentication)
- **Ingress Controller** (e.g., Traefik, NGINX Ingress Controller)

---

## Architecture Overview

The 3Sixty stack consists of the following components deployed as Helm sub-charts:

| Component | Description | Version |
|-----------|-------------|---------|
| **threesixty** | Admin & Discovery web applications + RabbitMQ + SCIM server | 5.0.3 |
| **mongo** | MongoDB database for data persistence | 8.0.11 |
| **elasticsearch** | Single-node Elasticsearch cluster | 7.17.0 |
| **opensearch** | OpenSearch + Dashboards for search and analytics | 2.4.1 |
| **hybridsearch** | Ollama LLM server + Objective RAG agent | 0.7.0+5.0.1-RC3 |

---

## Configuration

### Main Chart Configuration

The main `values.yaml` file allows you to enable/disable individual components:

```yaml
# Toggle each sub-chart on/off
mongo:
  enabled: true

threesixty:
  enabled: true

elasticsearch:
  enabled: true

opensearch:
  enabled: true

hybridsearch:
  enabled: true

# Global settings (shared across all sub-charts)
global:
  imagePullSecrets:
    - name: ecr-registry-secret
```

### Component-Specific Configuration

Each sub-chart has its own `values.yaml` file in the `charts/` directory:

- `charts/threesixty/values.yaml` - Admin, Discovery, RabbitMQ, and SCIM configuration
- `charts/mongo/values.yaml` - MongoDB settings
- `charts/elasticsearch/values.yaml` - Elasticsearch configuration
- `charts/opensearch/values.yaml` - OpenSearch and Dashboards settings
- `charts/hybridsearch/values.yaml` - Ollama and RAG agent configuration

### Security Best Practices

**⚠️ IMPORTANT**: Never commit sensitive credentials to version control. The chart now uses Kubernetes Secrets for all sensitive data.

#### Sensitive Configuration

All sensitive data is now stored in Kubernetes Secrets:

- **MongoDB credentials** - Stored in `mongodb-secret`
- **RabbitMQ credentials** - Stored in `admin-secret`
- **OAuth2 credentials** - Stored in `discovery-secret`
- **SCIM credentials** - Stored in `scim-secret`

#### Automatic Connection String Generation

The `MONGODB_URI` is now automatically generated from individual components:
- Username and password from `mongodb.username` and `mongodb.password`
- Host from the MongoDB service name
- Database from `mongodb.database`

#### Example Secure Values File

Use `values-secure.yaml` as a template for production deployments:

```bash
# Copy the secure template
cp charts/threesixty/values-secure.yaml charts/threesixty/my-production-values.yaml

# Edit with your actual credentials
vim charts/threesixty/my-production-values.yaml

# Deploy with secure values
helm install threesixty-stack . -n threesixty -f charts/threesixty/my-production-values.yaml
```

#### External Secrets Management

For production environments, consider using external secrets management solutions:

- **HashiCorp Vault**
- **AWS Secrets Manager**
- **Azure Key Vault**
- **External Secrets Operator**

Example with External Secrets Operator:
```yaml
# Create external secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: threesixty-mongodb-secret
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: threesixty-stack-mongodb-secret
  data:
    - secretKey: MONGODB_PASSWORD
      remoteRef:
        key: threesixty/mongodb/password
```

---

## AWS ECR Authentication

Before deploying, ensure you're authenticated with AWS ECR to pull the necessary Docker images:

```bash
aws configure
# Enter your credentials and region when prompted:
# AWS Access Key ID [None]: <your-access-key-id>
# AWS Secret Access Key [None]: <your-secret-access-key>
# Default region name [None]: ap-southeast-2
# Default output format [None]: json

aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin 782396859527.dkr.ecr.ap-southeast-2.amazonaws.com
```

Create a Kubernetes secret for ECR authentication:

```bash
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=782396859527.dkr.ecr.ap-southeast-2.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-southeast-2)
```

---

## Deployment

### 1. Install Dependencies

First, update and install the chart dependencies:

```bash
helm dependency update
```

### 2. Deploy the Stack

Deploy all components:

```bash
helm install threesixty-stack . -n <namespace>
```

Or deploy to a new namespace:

```bash
kubectl create namespace threesixty
helm install threesixty-stack . -n threesixty
```

### 3. Deploy with Custom Values

To override default values:

```bash
helm install threesixty-stack . -n threesixty -f custom-values.yaml
```

### 4. Upgrade Existing Deployment

```bash
helm upgrade threesixty-stack . -n threesixty
```

---

## Service Endpoints

After deployment, the following services will be available:

| Service | Internal Endpoint | External Access |
|---------|------------------|-----------------|
| Admin UI | `http://threesixty-stack-admin:8080/3sixty-admin` | `https://localhost/3sixty-admin` |
| Discovery UI | `http://threesixty-stack-discovery:8080` | `https://localhost/3sixty-discovery` |
| SCIM Server | `http://threesixty-stack-scim-server:8083` | `https://localhost/scim/v2` |
| RabbitMQ Management | `http://threesixty-stack-rabbitmq:15672` | ClusterIP Only |
| OpenSearch | `http://threesixty-stack-opensearch:9200` | ClusterIP Only |
| OpenSearch Dashboards | `http://threesixty-stack-opensearch-dashboard:5601` | ClusterIP Only |
| Elasticsearch | `http://threesixty-stack-elasticsearch:9200` | ClusterIP Only |
| Ollama | `http://threesixty-stack-hybridsearch-ollama:11434` | ClusterIP Only |
| RAG Agent | `http://threesixty-stack-hybridsearch-oirag:8080` | ClusterIP Only |

---

## Ingress Configuration

The threesixty chart includes an Ingress resource configured for Traefik. The following services are accessible externally via HTTPS:

### Currently Accessible Services

- **Admin UI:** `https://localhost/3sixty-admin`
- **Discovery UI:** `https://localhost/3sixty-discovery`
- **SCIM Server:** `https://localhost/scim/v2`

### Current TLS Configuration

The ingress is configured with TLS for secure access:

```yaml
ingress:
  ingressClassName: traefik
  tls:
    - secretName: nginx-tls
      hosts:
        - localhost
```

### Custom Domain Setup

To use a custom domain instead of localhost, update the ingress configuration in `charts/threesixty/values.yaml`:

```yaml
ingress:
  ingressClassName: traefik
  tls:
    - secretName: your-tls-secret
      hosts:
        - your-domain.com
```

---

## Environment Variables

### Non-Sensitive Configuration (ConfigMaps)

These environment variables are stored in ConfigMaps and are safe to commit to version control:

```yaml
env:
  admin:
    APP_URI: "/3sixty-admin"
    RABBITMQ_HOST: "threesixty-stack-rabbitmq"
    RABBITMQ_QUEUE: "scim-queue"
    GRPC_SERVER_START: "true"
    GRPC_SERVER_SSL: "false"
    GLOBAL_ORG: "objective"

  discovery:
    APP_URI: "http://threesixty-stack-admin:8080/3sixty-admin"
    OAUTH2_ENABLED: "true"
```

### Sensitive Configuration (Secrets)

These environment variables are automatically generated and stored in Kubernetes Secrets:

#### MongoDB Configuration
```yaml
mongodb:
  username: "dbuser"
  password: "your-secure-password"  # Stored in mongodb-secret
  database: "dbtest"
  scimDatabase: "scim-database"
```

**Automatically generates:**
- `MONGODB_URI`: `mongodb://username:password@service:27017/database`
- `MONGODB_USERNAME`, `MONGODB_PASSWORD`, `MONGODB_DATABASE`

#### OAuth2 Configuration
```yaml
oauth2:
  clientId: "your-azure-client-id"
  tenantId: "your-azure-tenant-id"
  clientSecret: "your-azure-client-secret"
```

**Automatically generates:**
- `CLIENT_ID`, `TENANT_ID`, `CLIENT_SECRET` (stored in discovery-secret)

#### RabbitMQ Configuration
```yaml
rabbitmq:
  username: "rabbitmq-user"
  password: "your-secure-password"
```

**Automatically generates:**
- `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD` (stored in admin-secret)

### RAG Agent Configuration

The hybridsearch component includes Ollama and the RAG agent:

```yaml
oirag:
  env:
    OPENAI_API_BASE: "http://threesixty-stack-hybridsearch-ollama:11434/v1"
    OPENSEARCH_URL: "http://threesixty-stack-opensearch:9200"
    EMBEDDING_MODEL: "mxbai-embed-large"
    QA_MODEL: "mxbai-embed-large"
```

---

## Persistence

### MongoDB

MongoDB data is persisted using PersistentVolumeClaims. Configure storage class and size in `charts/mongo/values.yaml`.

### OpenSearch

OpenSearch data persistence is configured in `charts/opensearch/values.yaml`.

### Ollama Models

Ollama models are persisted using a PVC with 5Gi storage by default. Configure in `charts/hybridsearch/values.yaml`.

---

## Monitoring and Logging

### Service Health Checks

All services include health check endpoints and readiness/liveness probes.

### Logs

Access logs for individual pods:

```bash
kubectl logs -f deployment/threesixty-stack-admin -n threesixty
kubectl logs -f deployment/threesixty-stack-discovery -n threesixty
kubectl logs -f deployment/threesixty-stack-hybridsearch-oirag -n threesixty
```

### OpenSearch Dashboards

Access OpenSearch Dashboards for search analytics and monitoring:

```bash
kubectl port-forward svc/threesixty-stack-opensearch-dashboard 5601:5601 -n threesixty
```

Then visit `http://localhost:5601`

---

## Troubleshooting

### Common Issues

1. **Image Pull Errors**
   - Ensure ECR authentication is properly configured
   - Verify the `ecr-registry-secret` exists in your namespace

2. **Service Communication Issues**
   - Check that all services are running: `kubectl get pods -n threesixty`
   - Verify network policies allow inter-service communication

3. **Database Connection Issues**
   - Ensure MongoDB is running and accessible
   - Check MongoDB credentials in environment variables

4. **Ingress Not Working**
   - Verify your ingress controller is installed and running
   - Check ingress configuration and TLS certificates

### Debug Commands

```bash
# Check pod status
kubectl get pods -n threesixty

# Describe pod for detailed information
kubectl describe pod <pod-name> -n threesixty

# Check service endpoints
kubectl get endpoints -n threesixty

# Check ingress status
kubectl get ingress -n threesixty
```

---

## Uninstallation

To remove the entire stack:

```bash
helm uninstall threesixty-stack -n threesixty
```

**Note**: This will remove all deployments, services, and PVCs. Data in persistent volumes will be lost unless you manually preserve it.

---

## Security Considerations

### Production Deployment

For production environments, consider:

1. **TLS Certificates**: Use proper TLS certificates for all external endpoints
2. **Network Policies**: Implement network policies to restrict inter-service communication
3. **Secrets Management**: Use Kubernetes secrets or external secret management solutions
4. **RBAC**: Configure appropriate RBAC policies
5. **Resource Limits**: Set appropriate resource requests and limits for all pods

### SCIM Integration

For SCIM user provisioning, ensure your SCIM endpoint is accessible from Microsoft Entra ID (Azure AD) and configure the appropriate callback URLs.

---

## Support

For issues related to:
- **Helm Charts**: Check the individual chart README files in the `charts/` directory
- **3Sixty Services**: Refer to the service-specific documentation
- **Kubernetes**: Consult Kubernetes documentation and your cluster provider's support
