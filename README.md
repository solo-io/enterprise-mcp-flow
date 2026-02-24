# MCP Auth Demo (Eager Auth with Entra ID)

This deploys an Enterprise AgentGateway setup with MCP servers (GitHub, Atlassian, GitLab) fronted by Microsoft Entra ID authentication.

## Prerequisites

1. A real domain with DNS pointing to your cluster (OAuth redirects require a publicly reachable URL)
2. Follow [ENTRA.md](ENTRA.md) to set up the Entra login
3. Follow [MCP_GITHUB.md](MCP_GITHUB.md) to set up the GitHub OAuth app
4. Follow [MCP_ATLASSIAN.md](MCP_ATLASSIAN.md) to set up the Atlassian MCP server
5. Follow [MCP_GITLAB.md](MCP_GITLAB.md) to set up the GitLab MCP server

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

3. Install Enterprise AgentGateway. Create a values file (e.g. `agentgateway-values.yaml`) with your settings. Replace all `YOUR_*` placeholders:

```yaml
licensing:
  licenseKey: YOUR_LICENSE_KEY
tokenExchange:
  enabled: true
  issuer: "enterprise-agentgateway.agentgateway-system.svc.cluster.local:7777"
  tokenExpiration: 24h
  subjectValidator:
    validatorType: remote
    remoteConfig:
      url: "https://login.microsoftonline.com/YOUR_ENTRA_TENANT_ID/discovery/v2.0/keys"
  actorValidator:
    validatorType: k8s
  apiValidator:
    validatorType: remote
    remoteConfig:
      url: http://solo-enterprise-ui.agentgateway-system.svc.cluster.local:5556/keys
  database:
    type: postgres
    postgres:
      url: postgres://myuser:mypassword@postgres.postgres:5432/mydb
controller:
  extraEnv:
    KGW_OAUTH_ISSUER_CONFIG: |
      {
        "gateway_config": {
          "base_url": "YOUR_EXTERNAL_ADDRESS/oauth-issuer"
        },
        "downstream_server": {
          "name": "downstream",
          "client_id": "YOUR_ENTRA_CLIENT_ID",
          "client_secret": "YOUR_ENTRA_CLIENT_SECRET",
          "authorize_url": "https://login.microsoftonline.com/YOUR_ENTRA_TENANT_ID/oauth2/v2.0/authorize",
          "token_url": "https://login.microsoftonline.com/YOUR_ENTRA_TENANT_ID/oauth2/v2.0/token",
          "scopes": [
            "api://YOUR_ENTRA_CLIENT_ID/agentgateway"
          ]
        }
      }
```

```bash
helm upgrade -i --create-namespace --namespace agentgateway-system enterprise-agentgateway \
        PATH_TO_ENTERPRISE_AGENTGATEWAY_CHART \
        -f agentgateway-values.yaml
```

Where `PATH_TO_ENTERPRISE_AGENTGATEWAY_CHART` is the path to the `enterprise-agentgateway` Helm chart (e.g. a local checkout at `agentgateway-enterprise/ent-controller/install/generated/enterprise-agentgateway`).

4. Install the base gateway resources:

```bash
helm upgrade -i base-mcp-gateway -n default mcp-auth-demo/base \
        --set gateway.name=mcp
```

5. Install each MCP server using the `external-mcp` chart with inline values. Replace `YOUR_*` placeholders with your values.

**GitLab** (DCR — uses `baseUrl`):

```bash
cat <<EOF | helm upgrade -i gitlab -n default mcp-auth-demo/external-mcp -f -
gateway:
  name: mcp
  externalAddress: YOUR_EXTERNAL_ADDRESS
entra:
  tenantId: YOUR_ENTRA_TENANT_ID
  clientId: YOUR_ENTRA_CLIENT_ID
path: /mcp/gitlab
mcp:
  host: gitlab.com
  path: /api/v4/mcp
auth:
  baseUrl: https://gitlab.com
  scopes: [mcp]
EOF
```

**Atlassian** (DCR — uses `baseUrl`):

```bash
cat <<EOF | helm upgrade -i atlassian -n default mcp-auth-demo/external-mcp -f -
gateway:
  name: mcp
  externalAddress: YOUR_EXTERNAL_ADDRESS
entra:
  tenantId: YOUR_ENTRA_TENANT_ID
  clientId: YOUR_ENTRA_CLIENT_ID
path: /mcp/atlassian
mcp:
  host: mcp.atlassian.com
  path: /v1/mcp
auth:
  baseUrl: https://mcp.atlassian.com
  scopes: [read:jira-work]
EOF
```

**GitHub** (static credentials — uses `clientId`/`clientSecret`/`authorizeUrl`/`tokenUrl`):

```bash
cat <<EOF | helm upgrade -i github -n default mcp-auth-demo/external-mcp -f -
gateway:
  name: mcp
  externalAddress: YOUR_EXTERNAL_ADDRESS
entra:
  tenantId: YOUR_ENTRA_TENANT_ID
  clientId: YOUR_ENTRA_CLIENT_ID
path: /mcp/github
mcp:
  host: api.githubcopilot.com
auth:
  clientId: YOUR_GITHUB_CLIENT_ID
  clientSecret: YOUR_GITHUB_CLIENT_SECRET
  authorizeUrl: https://github.com/login/oauth/authorize
  tokenUrl: https://github.com/login/oauth/access_token
  scopes: [repo, read:org]
EOF
```

To preview the rendered manifests without installing, replace `helm upgrade -i` with `helm template` in any of the commands above.

### With helmfile

If you have [helmfile](https://github.com/helmfile/helmfile) installed, steps 3–5 above can be replaced with a single command.

First, populate `common-values.yaml` and `client-secrets.yaml`:

```bash
cp common-values.yaml.example common-values.yaml
cp client-secrets.yaml.example client-secrets.yaml
```

Edit both files with your real values:

- **`common-values.yaml`** — gateway name, external address, Entra tenant/client IDs. Update `gateway.externalAddress` to match your domain.
- **`client-secrets.yaml`** — license key, Entra client secret, GitHub OAuth credentials. Git-ignored.

Then deploy:

```bash
helmfile sync --file helmfile.yaml.gotmpl
```

The helmfile reads secrets from `client-secrets.yaml` and shared config from `common-values.yaml`, so you don't need to pass them inline.

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

GitHub, Gitlab and Atlassian are already configured. To add a different MCP server, use the `external-mcp` chart.

For MCP servers that support Dynamic Client Registration (DCR), use `baseUrl`:

```bash
cat <<EOF | helm upgrade -i my-server -n default mcp-auth-demo/external-mcp -f -
gateway:
  name: mcp
  externalAddress: YOUR_EXTERNAL_ADDRESS
entra:
  tenantId: YOUR_ENTRA_TENANT_ID
  clientId: YOUR_ENTRA_CLIENT_ID
path: /mcp/my-server
mcp:
  host: mcp.example.com
  path: /v1/mcp
auth:
  baseUrl: https://mcp.example.com
  scopes: [read:data]
EOF
```

For MCP servers that require static client credentials (like GitHub), specify `clientId`, `clientSecret`, `authorizeUrl`, and `tokenUrl` instead of `baseUrl`:

```bash
cat <<EOF | helm upgrade -i my-server -n default mcp-auth-demo/external-mcp -f -
gateway:
  name: mcp
  externalAddress: YOUR_EXTERNAL_ADDRESS
entra:
  tenantId: YOUR_ENTRA_TENANT_ID
  clientId: YOUR_ENTRA_CLIENT_ID
path: /mcp/my-server
mcp:
  host: api.example.com
auth:
  clientId: YOUR_CLIENT_ID
  clientSecret: YOUR_CLIENT_SECRET
  authorizeUrl: https://example.com/oauth/authorize
  tokenUrl: https://example.com/oauth/token
  scopes: [repo]
EOF
```

If using helmfile, add a new release to `helmfile.yaml.gotmpl` following the same pattern (the `common-values.yaml` file provides the shared `gateway` and `entra` values).

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
