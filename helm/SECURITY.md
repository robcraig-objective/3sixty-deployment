# Security Best Practices for 3Sixty Helm Charts

This document outlines the security improvements and best practices implemented in the 3Sixty Helm charts.

## 🔒 Security Improvements

### Before (Security Issues)
- ❌ Credentials stored in plain text in `values.yaml`
- ❌ Hardcoded `MONGODB_URI` with duplicated information
- ❌ Sensitive data committed to version control
- ❌ Inconsistent secret handling across services

### After (Security Best Practices)
- ✅ All sensitive data moved to Kubernetes Secrets
- ✅ Automatic connection string generation
- ✅ Clear separation of sensitive vs non-sensitive configuration
- ✅ Consistent secret management across all services

## 🛡️ Secrets Management

### Kubernetes Secrets Created

| Secret Name | Purpose | Contents |
|-------------|---------|----------|
| `mongodb-secret` | MongoDB credentials | `MONGODB_USERNAME`, `MONGODB_PASSWORD`, `MONGODB_DATABASE`, `MONGODB_SCIM_DATABASE` |
| `admin-secret` | Admin service secrets | `MONGODB_URI`, `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD`, `SCIM_TOKEN_URL`, `SCIM_USERNAME`, `SCIM_PASSWORD` |
| `discovery-secret` | OAuth2 credentials | `CLIENT_ID`, `TENANT_ID`, `CLIENT_SECRET` |
| `scim-secret` | SCIM service secrets | `SCIM_USERNAME`, `SCIM_PASSWORD`, `SCIM_STORAGE_MONGO_CONSTR`, `SCIM_STORAGE_MONGO_DATABASE` |

### Automatic Connection String Generation

The `MONGODB_URI` is now automatically generated from individual components:

```yaml
# Input configuration
mongodb:
  username: "dbuser"
  password: "secure-password"
  database: "dbtest"

# Automatically generates:
MONGODB_URI: "mongodb://dbuser:secure-password@threesixty-stack-mongo:27017/dbtest"
```

## 📝 Configuration Structure

### Non-Sensitive Configuration (ConfigMaps)
```yaml
env:
  admin:
    APP_URI: "/3sixty-admin"
    RABBITMQ_HOST: "threesixty-stack-rabbitmq"
    GLOBAL_ORG: "objective"
```

### Sensitive Configuration (Secrets)
```yaml
mongodb:
  username: "dbuser"
  password: "your-secure-password"  # Never commit this!

oauth2:
  clientId: "your-azure-client-id"
  clientSecret: "your-azure-client-secret"  # Never commit this!
```

## 🚀 Production Deployment

### 1. Create Secure Values File
```bash
cp charts/threesixty/values-secure.yaml charts/threesixty/production-values.yaml
```

### 2. Edit with Real Credentials
```yaml
mongodb:
  username: "prod-dbuser"
  password: "super-secure-production-password"
  
oauth2:
  clientId: "your-actual-azure-client-id"
  clientSecret: "your-actual-azure-client-secret"
```

### 3. Deploy with Secure Values
```bash
helm install threesixty-stack . -n threesixty -f charts/threesixty/production-values.yaml
```

## 🔐 External Secrets Management

For enterprise environments, use external secrets management:

### HashiCorp Vault
```yaml
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

### AWS Secrets Manager
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: threesixty-oauth2-secret
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: threesixty-stack-discovery-secret
  data:
    - secretKey: CLIENT_SECRET
      remoteRef:
        key: threesixty/oauth2/client-secret
```

## 🔍 Security Checklist

Before deploying to production:

- [ ] All passwords are strong and unique
- [ ] OAuth2 credentials are properly configured
- [ ] MongoDB credentials are secure
- [ ] RabbitMQ credentials are changed from defaults
- [ ] TLS certificates are properly configured
- [ ] Network policies are in place
- [ ] RBAC is properly configured
- [ ] Resource limits are set
- [ ] Secrets are not committed to version control
- [ ] External secrets management is considered

## 🚨 Security Warnings

### Never Do This:
```yaml
# ❌ DON'T - Credentials in plain text
mongodb:
  password: "password123"
```

```bash
# ❌ DON'T - Commit secrets to Git
git add values.yaml  # If it contains passwords
```

### Always Do This:
```yaml
# ✅ DO - Use strong passwords
mongodb:
  password: "xK9#mP2$vL8@nQ4!jR7"

# ✅ DO - Use external secrets
# Store sensitive data in Vault/AWS Secrets Manager
```

## 🔧 Troubleshooting

### Check Secrets
```bash
# List all secrets
kubectl get secrets -n threesixty

# Check secret contents (base64 encoded)
kubectl get secret threesixty-stack-mongodb-secret -n threesixty -o yaml
```

### Verify Secret Mounting
```bash
# Check if secrets are mounted in pods
kubectl describe pod <pod-name> -n threesixty

# Check environment variables in running container
kubectl exec <pod-name> -n threesixty -- env | grep MONGODB
```

## 📚 Additional Resources

- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [External Secrets Operator](https://external-secrets.io/)
- [HashiCorp Vault](https://www.vaultproject.io/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/) 
