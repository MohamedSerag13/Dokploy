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
