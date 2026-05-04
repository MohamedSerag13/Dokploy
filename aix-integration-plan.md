# AIX to Dokploy Integration Plan

This document outlines the technical specification for automatically deploying apps via Dokploy when a customer makes a purchase on the AIX Marketplace.

## Overview Flow

1. **Trigger:** A customer successfully completes a purchase in the AIX store.
2. **Webhook:** The AIX backend receives the purchase confirmation and initiates the deployment workflow.
3. **API Orchestration:** The AIX backend calls the Dokploy REST API to create a new application deployment under the customer's project.
4. **Configuration:** The deployment source is the seller's repository. The Dokploy deployment is configured based on the `aix.deploy.yaml` spec found in that repo, which defines the runtime, build commands, start commands, and required environment variables.
5. **Provisioning:** Dokploy provisions the Docker container and assigns a Traefik routing rule using the subdomain.
6. **Delivery:** The AIX backend stores the resulting URL (`<app-slug>-<tenant-id>.aix.codemeld.io`) and presents it to the customer on their dashboard.

## Sample API Call Sequence

The following pseudocode/cURL examples demonstrate the sequence of REST API calls the AIX backend will make to the Dokploy API.

### 1. Create a Project
Group deployments by tenant/customer.

```bash
curl -X POST "$DOKPLOY_URL/api/projects" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "tenant-12345",
    "description": "Project for Tenant 12345"
  }'
```

### 2. Create the Application
Create the app within the newly created project using the seller's repository.

```bash
curl -X POST "$DOKPLOY_URL/api/applications" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "projectId": "<project-id-from-step-1>",
    "name": "teamflow",
    "repository": "https://github.com/seller/teamflow.git",
    "branch": "main",
    "buildType": "docker",
    "env": "PORT=3000\nNODE_ENV=production"
  }'
```
*(Note: The build configuration will be dynamically parsed from the seller's `aix.deploy.yaml` before making this call.)*

### 3. Assign a Subdomain
Map the customer's dedicated subdomain to the application so Traefik can route traffic appropriately.

```bash
curl -X POST "$DOKPLOY_URL/api/applications/<application-id>/domains" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "host": "teamflow-12345.aix.codemeld.io",
    "https": true
  }'
```

### 4. Trigger Build and Deploy
Initiate the deployment process. Dokploy will pull the code, build the container, and start it.

```bash
curl -X POST "$DOKPLOY_URL/api/applications/<application-id>/deploy" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN"
```
