# Auth demo

```bash
helmfile sync --file demo/helmfile.yaml.gotmpl
```

Replace the settings in `helmfile.yaml.gotmpl` and `common-values.yaml`.
For each MCP server, you will need to modify both of them.

Additionally, make a `client-secrets.yaml` as needed with values like:

```yaml
license: YOUR_LICENSE_KEY
entra:
  tenantId: YOUR_ENTRA_TENANT_ID
  id: YOUR_ENTRA_CLIENT_ID
  secret: YOUR_ENTRA_CLIENT_SECRET
github:
  id: YOUR_GITHUB_CLIENT_ID
  secret: YOUR_GITHUB_CLIENT_SECRET
```

