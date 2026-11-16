# obsidian-mcp

k3s deployment for a remote-accessible Obsidian vault MCP server. One pod,
three containers:

- **sync** — `ghcr.io/crosbyh/obsidian-headless-sync-docker`, keeps the vault
  in sync via Obsidian Sync
- **web-mcp** — [obsidian-web-mcp](https://github.com/jimprosser/obsidian-web-mcp),
  MCP-over-HTTPS server on the vault; built from `./obsidian-web-mcp` and
  pushed to `ghcr.io/bigreds/obsidian-web-mcp`
- **cloudflared** — Cloudflare Tunnel, the only ingress

Traffic: internet → cloudflared → web-mcp over pod loopback (127.0.0.1:8420) →
vault volume shared with sync. No Service, no inbound ports.

## Secrets and credentials

All runtime secrets are in `secret.yaml` (applied by kustomize as the
`obsidian-mcp` Secret). **It is gitignored — never commit real values.**
On a fresh clone: `cp secret.example.yaml secret.yaml` and fill it in.

| Key | Where it comes from |
|-----|---------------------|
| `OBSIDIAN_AUTH_TOKEN` | `podman run --rm -it ghcr.io/crosbyh/obsidian-headless-sync-docker:latest get-token` (interactive: email, password, MFA) |
| `VAULT_NAME` | Exact, case-sensitive remote vault name. List yours: `podman run --rm -e OBSIDIAN_AUTH_TOKEN=... ghcr.io/crosbyh/obsidian-headless-sync-docker:latest ob sync-list-remote` |
| `VAULT_PASSWORD` | Only if the vault has E2E encryption (Obsidian → Settings → Sync) |
| `VAULT_MCP_TOKEN` | `python3 -c "import secrets; print(secrets.token_hex(32))"` — bearer token for MCP clients |
| `VAULT_OAUTH_PASSWORD` | Passphrase you invent; required for the OAuth browser login (`/oauth/authorize`) |
| `TUNNEL_TOKEN` | Cloudflare Zero Trust → Networks → Tunnels → your tunnel |

Machine setup (only for local skaffold builds; GHA needs nothing):

```bash
podman login ghcr.io                          # PAT with write:packages
cp /run/user/$(id -u)/containers/auth.json ~/.docker/config.json
chmod 600 ~/.docker/config.json               # skaffold reads creds from here, not podman's
```

## Dev

Builds run on GitHub Actions (`.github/workflows/build.yaml`), multi-arch
(amd64+arm64), only when `obsidian-web-mcp/` changes:

- push to `main` → `:latest`
- push to `develop` → `:dev`
- both also tagged `:<branch>-<sha>`

Deploy from any clone with a kubecontext:

```bash
cp secret.example.yaml secret.yaml   # then fill in real values — see table above
kubectl -k .
```

(kustomize needs `secret.yaml`, which is gitignored — the deploy fails without
it.) The manifests reference `:latest`; to deploy the dev build instead:
`kustomize edit set image ghcr.io/cubanyetis/obsidian-mcp-k8s=ghcr.io/cubanyetis/obsidian-mcp-k8s:dev`
before `kubectl -k .`.

`skaffold.yaml` remains for optional local builds (`skaffold run`), but those
are single-arch for the build host — arm64 on merlin, so useless for amd64
nodes. Prefer pushing to `develop`.

Useful:

```bash
skaffold delete                                                  # tear down
kubectl -n obsidian-mcp logs deploy/obsidian-mcp -c sync         # or web-mcp / cloudflared
```

Troubleshooting: `ImagePullBackOff` on the web-mcp image usually means the
GHCR package defaulted to private — flip it public under
github.com → Packages → obsidian-web-mcp → Package settings, or add
`imagePullSecrets`.
