# Atlassian MCP Setup

This documents how to configure the Atlassian MCP server as an upstream for the agentgateway demo.

## Prerequisites

- An Atlassian Cloud site (e.g. `yoursite.atlassian.net`)
- **Organization admin** access to the Atlassian site
- The agentgateway already installed (see README.md)
- Your gateway hostname e.g. `https://mcp.k8s.example.com`

## Steps

### 1. Register the gateway domain in Rovo MCP server settings

The Atlassian MCP server requires your gateway's domain to be explicitly registered before users can complete the OAuth consent flow.

1. Go to **[admin.atlassian.com](https://admin.atlassian.com)** and select your organization
2. Navigate to **Apps → AI settings → Rovo MCP server**
3. Add your gateway domain with a path wildcard e.g

```
https://mcp.k8s.example.com/**
```

The `/**` path wildcard is required — it allows Atlassian to redirect to the OAuth callback paths served by the gateway.

Atlassian uses [OAuth 2.0 Dynamic Client Registration](https://tools.ietf.org/html/rfc7591), so no manual OAuth app creation is needed and no additional configuration is required — the Atlassian MCP server is already configured in `helmfile.yaml.gotmpl` using values from `common-values.yaml`.

### 2. Verify and connect

After adding the domain, retry the OAuth consent flow. The warning "Your organization admin must authorize access from a domain to this site" should no longer appear, and users can grant the requested Jira and Confluence permissions.

## Troubleshooting

### "Your organization admin must authorize access from a domain to this site"

This error appears on the Atlassian OAuth consent page when the gateway domain hasn't been registered in the Rovo MCP server settings.

**Fix:** Follow step 1 above. Make sure the domain includes the `/**` path wildcard.
