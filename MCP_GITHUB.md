# GitHub OAuth Setup

This documents how to configure GitHub OAuth for the agentgateway MCP demo.

## Prerequisites

- A GitHub account with access to create OAuth Apps
- The agentgateway already installed (see README.md)
- Your gateway hostname: `https://mcp.k8s.example.com`

## Steps

### 1. Create a GitHub OAuth App

Go to **GitHub → Settings → Developer settings → OAuth Apps → New OAuth App** and fill in:

| Field | Value |
|---|---|
| Application name | `agentgateway demo` (or anything descriptive) |
| Homepage URL | `https://mcp.k8s.example.com` |
| Authorization callback URL | `https://mcp.k8s.example.com/oauth-issuer/callback/upstream` |

### 2. Get your credentials

After creating the app, GitHub will show you:
- **Client ID** — copy this value
- **Client Secret** — click "Generate a new client secret" and copy it immediately (it won't be shown again)

### 3. Update `client-secrets.yaml`

Add your GitHub credentials to `client-secrets.yaml`:

```yaml
github:
  id: <YOUR_GITHUB_CLIENT_ID>
  secret: <YOUR_GITHUB_CLIENT_SECRET>
```

The helmfile template (`helmfile.yaml.gotmpl`) will use these values to populate the GitHub OAuth config automatically.

### 4. Redeploy

```bash
helmfile sync
```
