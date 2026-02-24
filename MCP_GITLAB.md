# GitLab MCP Setup

This documents how to configure GitLab as an upstream MCP server for the agentgateway demo.

## Prerequisites

- A GitLab.com account
- A GitLab **Premium or Ultimate** subscription (a [free Ultimate trial](https://about.gitlab.com/free-trial/) works)
- The agentgateway already installed (see README.md)
- Your gateway hostname e.g. `https://mcp.k8s.example.com`

## Steps

### 1. Create a top-level group (if you don't have one)

GitLab Duo settings are configured at the **top-level group** level. If you don't already have a group with Premium or Ultimate:

1. Click the **+** button in the GitLab top nav → **New group**
2. Give it a name and create it
3. Go to the group's **Settings → Billing** and start an Ultimate trial if needed

### 2. Enable GitLab Duo and beta features

The GitLab MCP server is a **Beta** feature and requires both Duo Core and beta features to be enabled.

1. Go to your top-level group → **Settings → GitLab Duo**
2. Click **Change configuration**
3. Ensure **GitLab Duo availability** is not set to "Always off"
4. Check **"Turn on features for GitLab Duo Core"**
5. Under **Feature preview**, check **"Turn on experiment and beta GitLab Duo features"**
6. Click **Save changes**

> **Note:** Changes can take up to 10 minutes to propagate.

### 3. Verify the MCP endpoint is active

Test the connection directly using `mcp-remote` (requires Node.js 20+):

```bash
rm -rf ~/.mcp-auth/mcp-remote*

npx -y mcp-remote@latest https://gitlab.com/api/v4/mcp \
  --static-oauth-client-metadata '{"scope": "mcp"}'
```

This will open a browser window for GitLab OAuth. After authorizing, a successful connection confirms the MCP feature is active. If it fails with a **404**, re-check step 2 and wait up to 10 minutes for settings to propagate.

GitLab uses [OAuth 2.0 Dynamic Client Registration](https://tools.ietf.org/html/rfc7591), so no manual OAuth app creation is needed and no additional configuration is required — the GitLab MCP server is already configured in `helmfile.yaml.gotmpl` using values from `common-values.yaml`.

## Troubleshooting

### 404 Not Found from GitLab

This means the MCP feature is not active for your user. Check:

- **Subscription tier** — you need Premium or Ultimate on your top-level group
- **GitLab Duo Core** — must be enabled in the group's Settings → GitLab Duo
- **Beta features** — must be enabled in the same settings page
- **Propagation delay** — wait up to an hour or two after changing settings
- **Session refresh** — try logging out and back in to GitLab

You can test directly (bypassing the gateway) with `mcp-remote`:

```bash
rm -rf ~/.mcp-auth/mcp-remote*
npx -y mcp-remote@latest https://gitlab.com/api/v4/mcp \
  --static-oauth-client-metadata '{"scope": "mcp"}' --debug
```

### 422 "session ID is required"

This is a secondary error. The gateway returns 422 when the VS Code client falls back to legacy SSE after the upstream 404. Fix the 404 (see above) and this error goes away.

### Clearing cached tokens in the gateway

If you change GitLab settings and need to force a fresh OAuth flow through the gateway, perform an MCP logout from your client (e.g. in VS Code, use the MCP server menu to log out of the GitLab server).
