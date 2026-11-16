# Required environment variables (no working defaults; set at runtime):
#   VAULT_PATH            - absolute path to the vault directory (mount it here: /vault)
#   VAULT_MCP_TOKEN       - 256-bit bearer token (python -c "import secrets; print(secrets.token_hex(32))")
#   VAULT_OAUTH_PASSWORD  - interactive login password; without it the server refuses all authorization

FROM python:3.12-slim

# Links the GHCR package to the source repo on push (github.com/<repo>/packages).
LABEL org.opencontainers.image.source=https://github.com/CubanYetis/obsidian-mcp-k8s

RUN apt-get update && apt-get install -y --no-install-recommends git \
 && pip install --no-cache-dir git+https://github.com/jimprosser/obsidian-web-mcp.git@v0.4.0 \
 && apt-get purge -y git && apt-get autoremove -y \
 && rm -rf /var/lib/apt/lists/*

ENV VAULT_PATH=/vault

# Non-root user; its HOME holds the OAuth state store (oauth_state.sqlite3),
# which the server keeps under ~/.local/share/vault-mcp.
RUN useradd --create-home --uid 1000 mcp
USER 1000

VOLUME /vault
EXPOSE 8420

CMD ["vault-mcp"]
