# 3Sixty Local Development

This repository contains everything you need to run all 3Sixty services locally via Docker Compose.

---

## Prerequisites

- **Docker** ≥ 20.10
- **Docker Compose** ≥ 1.29

---

## Configuration

_All `.env.*` files are excluded from Git. You’ll need to copy and rename the exising sample.env.* files first._

| Filename         | Purpose                           |
|------------------|-----------------------------------|
| `.env.admin`     | Environment for Admin webapp      |
| `.env.discovery` | Environment for Discovery service |
| `.env.oirag`     | Environment for RAG/Agent service |
| `.env.rabbitmq`  | RabbitMQ credentials & settings   |
| `.env.scim`      | SCIM server configuration         |
| `.env.oi-agent`  | Environment for OI remote agent   |

Place each file in the current folder before starting the stack.

You can leave the default values as they are. Make sure you set up values in `.env.discovery`
```
CLIENT_ID=
TENANT_ID=
CLIENT_SECRET=
```

Make sure to change `REMOTE_AGENT_TOKEN` and `REMOTE_AGENT_NAME` after a new remote agent is created.

To set the name of a shared docker volume use `name`:
```yaml
  mongo_data:
    name: docker_mongo_data
```

---

## Get a cert & key
To terminate TLS in Nginx you’ll need to obtain or generate a certificate + key.
For local testing you can self-sign:
```bash
mkdir nginx/certs
openssl req -x509 -newkey rsa:2048 -days 365 \
  -nodes \
  -keyout nginx/certs/tls.key \
  -out nginx/certs/tls.crt \
  -subj "/CN=localhost"
```

---

## Start Vault (Optional)

If you need to fetch secrets from Vault (e.g. `CLIENT_SECRET`), run the dev Vault server first:

```bash
docker compose -f docker-compose.vault.yml up -d
```

Listens on http://localhost:8200

Root token: root

## Launch All Services

Once your .env.* files are in place, simply:

```bash
docker compose up -d
```

Under the covers this will:
- nginx-proxy routes traffic on ports 80/443 using nginx/nginx.conf
- threesixty-admin and threesixty-discovery
- MongoDB, Elasticsearch, OpenSearch + Dashboard
- RabbitMQ & SCIM server for user provisioning
- Ollama + oi-rag (your RAG/agent service) on the `threesixty` network

After remote agent is configured (`.env.oi-agent`) start the OI Remote Agent:
```bash
docker compose -f docker-compose.oi-agent.yaml up -d
```

---

## Access URLs
| Service              | URL                                                                        |
|----------------------|----------------------------------------------------------------------------|
| Admin UI             | [https://localhost/3sixty-admin/](https://localhost/3sixty-admin/)         |
| Discovery UI         | [https://localhost/3sixty-discovery/](https://localhost/3sixty-discovery/) |
| SCIM API             | [https://localhost/scim/v2/](https://localhost/scim/v2/)                   |
| OpenSearch Dashboard | [http://localhost:15601](http://localhost:15601)                           |

Certificates are loaded from `./nginx/certs/tls.crt & ./nginx/certs/tls.key`
