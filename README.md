# MCP Auth Demo (Eager Auth with Entra ID)

This deploys an Enterprise AgentGateway setup with MCP servers (GitHub, Atlassian, GitLab) fronted by Microsoft Entra ID authentication, using helmfile.

## Prerequisites

1. A real domain with DNS pointing to your cluster (OAuth redirects require a publicly reachable URL)
2. [helmfile](https://github.com/helmfile/helmfile) installed
3. Follow [ENTRA.md](ENTRA.md) to set up the Entra login
4. Follow [MCP_GITHUB.md](MCP_GITHUB.md) to set up the GitHub OAuth app
5. Follow [MCP_ATLASSIAN.md](MCP_ATLASSIAN.md) to set up the Atlassian MCP server
6. Follow [MCP_GITLAB.md](MCP_GITLAB.md) to set up the GitLab MCP server

## Populating Secrets

Copy the example and fill in your values:

```bash
cp client-secrets.yaml.example client-secrets.yaml
```

Edit `client-secrets.yaml` with your real values. See the linked docs above for where to obtain them.

## Configuration

- **`common-values.yaml`** — shared values (gateway name, external address, Entra tenant/client IDs). Update `gateway.externalAddress` to match your domain.
- **`helmfile.yaml.gotmpl`** — helmfile that orchestrates all releases. To add a new MCP server, add a new `external-mcp` release following the existing pattern.
- **`client-secrets.yaml`** — secret values (license key, Entra secret, GitHub credentials). Git-ignored.
- **`mcp-auth-demo/`** — Helm charts for the base gateway infrastructure and per-MCP-server resources. Do not modify.

## Setup

1. Install the Gateway API CRDs and AgentGateway CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

helm upgrade -i --create-namespace --namespace agentgateway-system enterprise-agentgateway-crds \
        oci://us-central1-docker.pkg.dev/developers-369321/enterprise-agentgateway-public-nonprod/charts/enterprise-agentgateway-crds \
        --version v2.2.0-beta.3
```

2. Create the TLS secret from your cert and key files (see [Updating the TLS certificate](#updating-the-tls-certificate)):

```bash
kubectl create secret tls tls --cert=tls.crt --key=tls.key -n agentgateway-system
```

3. Deploy everything via helmfile:

```bash
helmfile sync --file helmfile.yaml.gotmpl
```

## Updating the TLS certificate

Generate a cert and key with [mkcert](https://github.com/filippo-io/mkcert):

```bash
brew install mkcert
mkcert -cert-file tls.crt -key-file tls.key localhost "*.k8s.example.com"
```

Then create (or update) the secret:

```bash
kubectl create secret tls tls --cert=tls.crt --key=tls.key -n default --dry-run=client -o yaml | kubectl apply -f -
```

Optionally, install the mkcert CA into your system trust store:

```bash
mkcert -install
```

## Adding a new MCP server

GitHub, Gitlab and Atlassian are already configured. To add a different MCP server add a new release to `helmfile.yaml.gotmpl`. For MCP servers that support Dynamic Client Registration (DCR), use `baseUrl`:

```yaml
  - name: my-server
    namespace: default
    chart: ./mcp-auth-demo/external-mcp
    values:
    - common-values.yaml
    - path: /mcp/my-server
      mcp:
        host: mcp.example.com
        path: /v1/mcp
      auth:
        baseUrl: https://mcp.example.com
        scopes: [read:data]
```

For MCP servers that require static client credentials (like GitHub), specify `clientId`, `clientSecret`, `authorizeUrl`, and `tokenUrl` instead of `baseUrl`:

```yaml
  - name: my-server
    namespace: default
    chart: ./mcp-auth-demo/external-mcp
    values:
    - common-values.yaml
    - path: /mcp/my-server
      mcp:
        host: api.example.com
      auth:
        clientId: '{{$secrets.myserver.id}}'
        clientSecret: '{{$secrets.myserver.secret}}'
        authorizeUrl: https://example.com/oauth/authorize
        tokenUrl: https://example.com/oauth/token
        scopes: [repo]
```

## Testing using VS Code

1. Open the Command Palette (`Cmd+Shift+P`)
2. Run **MCP: List MCP Servers**
3. For each endpoint, click **Add MCP Server**, choose **HTTP (Streamable)**, and enter the URL
4. Repeat for all three endpoints:
   - `https://mcp.k8s.example.com/mcp/github`
   - `https://mcp.k8s.example.com/mcp/atlassian`
   - `https://mcp.k8s.example.com/mcp/gitlab`

Each server will kick off a two-step login flow in your browser:

1. **Entra login** — authenticate with your Microsoft Entra identity
2. **MCP server login** — authorize access to the upstream service (GitHub, Atlassian, or GitLab)

## Testing using Cursor

1. Open **Cursor Settings** → **MCP**
2. Click **Add new MCP server** for each endpoint:
   - **Type:** HTTP (Streamable)
   - **URL:** the endpoint URL (see below)
3. Add all three servers:
   - `https://mcp.k8s.example.com/mcp/github`
   - `https://mcp.k8s.example.com/mcp/atlassian`
   - `https://mcp.k8s.example.com/mcp/gitlab`

The resulting `~/.cursor/mcp.json` should look like:

```json
{
  "mcpServers": {
    "github": {
      "url": "https://mcp.k8s.example.com/mcp/github"
    },
    "atlassian": {
      "url": "https://mcp.k8s.example.com/mcp/atlassian"
    },
    "gitlab": {
      "url": "https://mcp.k8s.example.com/mcp/gitlab"
    }
  }
}
```

Each server will kick off a two-step login flow in your browser:

1. **Entra login** — authenticate with your Microsoft Entra identity
2. **MCP server login** — authorize access to the upstream service (GitHub, Atlassian, or GitLab)

## Notes

- The `mcp-auth-demo/base/templates/common.yaml` chart has a hardcoded image registry (`localhost:5000`) in the `EnterpriseAgentgatewayParameters`. This may need to be parameterized for your deployment.
