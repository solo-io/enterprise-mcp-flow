# Microsoft Entra OAuth Setup

This documents how to configure Microsoft Entra ID (Azure AD) OAuth for the agentgateway MCP demo.

## Prerequisites

- Access to an Azure/Entra tenant where you can create App Registrations
- The agentgateway already installed (see README.md)
- Your gateway hostname: `https://mcp.k8s.example.com`
- Your tenant ID (found in Entra portal under **Tenant properties**)

## Steps

### 1. Create an App Registration

Go to **Azure Portal → Microsoft Entra ID → App registrations → New registration** and fill in:

| Field | Value |
|---|---|
| Name | `agentgateway demo` (or anything descriptive) |
| Supported account types | *Accounts in this organizational directory only* |
| Redirect URI (platform: **Web**) | `https://mcp.k8s.example.com/oauth-issuer/callback/downstream` |

Click **Register**.

### 2. Note your IDs

After registration, copy the values shown on the Overview page:

| Field | Where to find it |
|---|---|
| **Application (client) ID** | Overview page |
| **Directory (tenant) ID** | Overview page |

### 3. Create a Client Secret

Go to **Certificates & secrets → Client secrets → New client secret** and fill in:

| Field | Value |
|---|---|
| Description | `agentgateway` |
| Expires | Choose an appropriate expiry |

Click **Add**, then **immediately copy the secret Value** — it will not be shown again.

### 4. Expose an API and add a custom scope

Go to **Expose an API**:

1. Click **Add** next to *Application ID URI* — accept the default (`api://<client-id>`) and click **Save**.
2. Click **Add a scope** and fill in:

| Field | Value |
|---|---|
| Scope name | `agentgateway` |
| Who can consent | *Admins and users* |
| Admin consent display name | `Access agentgateway` |
| Admin consent description | `Allows the app to access agentgateway on behalf of the signed-in user.` |
| State | Enabled |

Click **Add scope**. The full scope will be `api://<client-id>/agentgateway`.

### 5. Update configuration files

Add your Entra IDs to `common-values.yaml`:

```yaml
entra:
  tenantId: <YOUR_TENANT_ID>
  clientId: <YOUR_CLIENT_ID>
```

Add your Entra credentials to `client-secrets.yaml`:

```yaml
entra:
  tenantId: <YOUR_TENANT_ID>
  id: <YOUR_CLIENT_ID>
  secret: <YOUR_CLIENT_SECRET>
```

The helmfile template (`helmfile.yaml.gotmpl`) will use these values to populate the downstream OAuth config and JWKS URL automatically.

### 6. Re-apply and redeploy

After updating the configuration files, re-run helmfile:

```bash
helmfile sync
```

## Troubleshooting

### AADSTS500113 — No reply address is registered

The redirect URI sent in the OAuth request doesn't match any registered URI on the app. Go to **App registrations → Authentication → Web → Redirect URIs** and ensure exactly this URI is listed:

```
https://mcp.k8s.example.com/oauth-issuer/callback/downstream
```

### AADSTS70011 — Invalid scope

The scope `api://<client-id>/agentgateway` must be defined under **Expose an API** (step 4). Check the Application ID URI matches the `client_id` prefix.

### AADSTS700016 — Application not found in tenant

The `client_id` doesn't match the app in the tenant specified by the `authorize_url` tenant ID. Verify the IDs in `common-values.yaml` and `client-secrets.yaml` are from the same app registration.
