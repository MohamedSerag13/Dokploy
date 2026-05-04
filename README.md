# Dokploy Infrastructure for AIX Marketplace

## What is Dokploy?
Dokploy is a self-hosted Platform as a Service (PaaS) built on Docker and Traefik. We chose Dokploy for the AIX Marketplace because:
- It is fully self-hosted, giving us complete control over our infrastructure.
- It has Traefik built-in for seamless routing, SSL termination, and reverse proxying.
- It is API-driven, which perfectly aligns with our automated provisioning goals.
- It natively supports wildcard subdomains, which is critical for providing customer-specific URLs (e.g., `teamflow-<tenant>.aix.codemeld.io`).

## Prerequisites
Before deploying Dokploy, you must configure a wildcard DNS record for your domain:
- Ensure that `*.aix.codemeld.io` points to the public IP address of the server hosting this Dokploy instance.

## How to Spin Up on a Fresh VPS
1. Clone this repository to your VPS.

3. Copy the example environment file and configure your variables:
   ```bash
   cp dokploy/.env.example dokploy/.env
   # Edit dokploy/.env with your specific values if needed
   ```
4. Start the Dokploy services in the background:
   ```bash
   docker compose up -d
   ```

## Next Integration Step
Once Dokploy is running, the next step is integrating it with the AIX Marketplace backend. 
When a customer purchases an app, the AIX purchase webhook will call Dokploy's REST API to automatically deploy the app. This flow involves reading an `aix.deploy.yaml` spec from the app seller's repository and assigning the customer a dedicated subdomain (e.g., `<app-slug>-<tenant-id>.aix.codemeld.io`). 

For a detailed technical overview of this API sequence, see [aix-integration-plan.md](./aix-integration-plan.md).

## Steps to Deploy Any App via Dokploy API

### One-time setup (already done)
```bash
docker swarm init --advertise-addr <SERVER_IP>
```

### Step 1 — Create a project
```bash
curl -X POST http://<HOST>:3000/api/project.create \
  -H "Content-Type: application/json" \
  -H "x-api-key: <API_KEY>" \
  -d '{"name": "my-project", "description": "optional"}'
```
*Save: `projectId`, `environmentId`*

### Step 2 — Create an application
```bash
curl -X POST http://<HOST>:3000/api/application.create \
  -H "Content-Type: application/json" \
  -H "x-api-key: <API_KEY>" \
  -d '{
    "name": "my-app",
    "projectId": "<projectId>",
    "environmentId": "<environmentId>"
  }'
```
*Save: `applicationId`*

### Step 3 — Set the Docker image
```bash
curl -X POST http://<HOST>:3000/api/application.update \
  -H "Content-Type: application/json" \
  -H "x-api-key: <API_KEY>" \
  -d '{
    "applicationId": "<applicationId>",
    "sourceType": "docker",
    "dockerImage": "image:tag"
  }'
```

### Step 4 — Add port mapping
```bash
curl -X POST http://<HOST>:3000/api/port.create \
  -H "Content-Type: application/json" \
  -H "x-api-key: <API_KEY>" \
  -d '{
    "applicationId": "<applicationId>",
    "targetPort": 80,
    "publishedPort": 8983,
    "protocol": "tcp"
  }'
```

### Step 5 — Deploy
```bash
curl -X POST http://<HOST>:3000/api/application.deploy \
  -H "Content-Type: application/json" \
  -H "x-api-key: <API_KEY>" \
  -d '{"applicationId": "<applicationId>"}'
```

### Step 6 — Verify
```bash
# Check status
curl -s "http://<HOST>:3000/api/application.one?applicationId=<applicationId>" \
  -H "x-api-key: <API_KEY>" \
  | grep -o '"applicationStatus":"[^"]*"'

# Test the app
curl -I http://<SERVER_IP>:<publishedPort>
```

### Required network (one-time only)
| Network | Driver | Flag |
|---------|--------|------|
| `dokploy-network` | overlay | `--attachable` |

*This network must exist before any deployment. If it's missing you'll get the `network dokploy-network not found` error.*

## Scaling: 100 Users, 100 Subdomains — No Ports Needed

This is exactly what domains/subdomains are for. Instead of exposing a port per user, you assign each app a subdomain and let Traefik handle routing.

**The flow:**
- `user1.yourdomain.com` → Traefik → `app-user1` container
- `user2.yourdomain.com` → Traefik → `app-user2` container
- `user3.yourdomain.com` → Traefik → `app-user3` container

All on port 80/443, no custom ports needed.

### How to assign a subdomain via API

After creating the app (Steps 1-3), instead of `port.create` do:

```bash
curl -X POST http://<HOST>:3000/api/domain.create \
  -H "Content-Type: application/json" \
  -H "x-api-key: <API_KEY>" \
  -d '{
    "applicationId": "<applicationId>",
    "host": "user1.yourdomain.com",
    "port": 80,
    "https": false,
    "path": "/"
  }'
```

Then deploy. Traefik will automatically route `user1.yourdomain.com` to that container.

### Summary

| Question | Answer |
| :--- | :--- |
| **Network per client?** | ❌ No — one `dokploy-network` for all |
| **100 ports for 100 users?** | ❌ No — use subdomains instead |
| **How routing works?** | Traefik reverse proxy by hostname |
| **Port exposed?** | Only 80/443 on the host |

### DNS requirement

You'll need a wildcard DNS record pointing to your server:
`*.yourdomain.com` → **A record** → `100.79.76.79`

That way every subdomain automatically resolves to your server and Traefik handles the rest.
