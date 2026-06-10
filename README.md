# Claude Code MCP Servers

Inventory of all [MCP](https://modelcontextprotocol.io) (Model Context Protocol) servers configured for [Claude Code](https://claude.com/claude-code) in the home-directory config on this machine.

- **Source of truth:** `~/.claude.json` → top-level `mcpServers` (user scope — available in every project)
- **Settings files:** `~/.claude/settings.json` and `~/.claude/settings.local.json` contain **no** MCP-related configuration (no enable/disable lists, no `mcp__*` permission rules)
- **Project-scoped servers:** none — no project in `~/.claude.json` registers its own `mcpServers`
- **Generated:** 2026-06-10

> 🔒 All credential values are redacted — `<REDACTED>` marks places where a real secret exists in the local config; only variable **names** are documented here. Internal URLs are masked with `<...-host>` placeholders.

## Overview

| Server | Transport | Runtime | Package / Image | Purpose |
|---|---|---|---|---|
| [`gitlab`](#gitlab) | stdio | `npx` | `@zereight/mcp-gitlab` | GitLab API — repos, MRs, issues, CI |
| [`grafana`](#grafana) | stdio | `docker` | `grafana/mcp-grafana` | Grafana — dashboards, Prometheus/Loki, OnCall, incidents |
| [`kubernetes`](#kubernetes) | stdio | `npx` | `mcp-server-kubernetes` | kubectl / helm operations against configured clusters |
| [`playwright`](#playwright) | stdio | `npx` | `@playwright/mcp` | Browser automation (navigate, click, screenshot, etc.) |
| [`vault-mcp-server`](#vault-mcp-server) | stdio | `docker` | `hashicorp/vault-mcp-server` | HashiCorp Vault — secrets, mounts, PKI |

All servers are stdio transport; none use HTTP/SSE endpoints or custom headers.

## Server details

### gitlab

GitLab MCP server ([@zereight/mcp-gitlab](https://www.npmjs.com/package/@zereight/mcp-gitlab)) pointed at a self-hosted GitLab. Read-only mode is explicitly **off**, so write operations (creating MRs, notes, etc.) are enabled.

```bash
claude mcp add --scope user gitlab \
  -e GITLAB_API_URL="https://<gitlab-host>/api/v4" \
  -e GITLAB_PERSONAL_ACCESS_TOKEN="<REDACTED>" \
  -e GITLAB_READ_ONLY_MODE=false \
  -- npx -y @zereight/mcp-gitlab
```

### grafana

Official Grafana MCP server run as a Docker container. Credentials are passed into the container via `-e` env passthrough. Register once per Grafana instance (separate name, URL, and service-account token each).

```bash
claude mcp add --scope user grafana \
  -e GRAFANA_URL="https://<grafana-host>" \
  -e GRAFANA_SERVICE_ACCOUNT_TOKEN="<REDACTED>" \
  -- docker run --rm -i -e GRAFANA_URL -e GRAFANA_SERVICE_ACCOUNT_TOKEN grafana/mcp-grafana -t stdio
```

### kubernetes

Kubernetes MCP server ([mcp-server-kubernetes](https://www.npmjs.com/package/mcp-server-kubernetes)) using a merged kubeconfig. Destructive tools are **disabled** via `ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS=true`.

```bash
claude mcp add --scope user kubernetes \
  -e ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS=true \
  -e KUBECONFIG="$HOME/.kube/config.merged" \
  -- npx -y mcp-server-kubernetes
```

### playwright

Official Playwright MCP server ([@playwright/mcp](https://www.npmjs.com/package/@playwright/mcp)) with default settings — no env vars, always pulls `@latest`.

```bash
claude mcp add --scope user playwright -- npx @playwright/mcp@latest
```

### vault-mcp-server

Official HashiCorp Vault MCP server run as a Docker container with env passthrough for the Vault address and token.

```bash
claude mcp add --scope user vault-mcp-server \
  -e VAULT_ADDR="https://<vault-host>" \
  -e VAULT_TOKEN="<REDACTED>" \
  -- docker run -i --rm -e VAULT_ADDR -e VAULT_TOKEN hashicorp/vault-mcp-server
```

## CLI commands

Claude Code manages MCP servers via `claude mcp`:

```bash
claude mcp list                        # list configured servers (health-checked)
claude mcp get <name>                  # details for one server
claude mcp add <name> -e KEY=val -- <command> [args...]
                                       # add a stdio server
claude mcp add --transport http <name> <url> --header "Authorization: Bearer ..."
                                       # add an HTTP server
claude mcp add-json <name> '<json>'    # add a server from a JSON blob
claude mcp remove <name>               # remove a server
claude mcp add-from-claude-desktop     # import servers from Claude Desktop (Mac/WSL)
claude mcp reset-project-choices       # reset approve/reject choices for project .mcp.json servers
claude mcp serve                       # run Claude Code itself as an MCP server
```

`add`/`add-json`/`remove` accept `--scope`: `user` (default for this inventory — stored in `~/.claude.json`, available in every project), `local` (per-project, private to this machine), or `project` (written to a shareable `.mcp.json` in the repo).

### Alternative: `add-json`

Each server from [Server details](#server-details) can equivalently be registered from its raw JSON config:

```bash
claude mcp add-json --scope user gitlab '{"type":"stdio","command":"npx","args":["-y","@zereight/mcp-gitlab"],"env":{"GITLAB_API_URL":"https://<gitlab-host>/api/v4","GITLAB_PERSONAL_ACCESS_TOKEN":"<REDACTED>","GITLAB_READ_ONLY_MODE":"false"}}'

claude mcp add-json --scope user grafana '{"type":"stdio","command":"docker","args":["run","--rm","-i","-e","GRAFANA_URL","-e","GRAFANA_SERVICE_ACCOUNT_TOKEN","grafana/mcp-grafana","-t","stdio"],"env":{"GRAFANA_URL":"https://<grafana-host>","GRAFANA_SERVICE_ACCOUNT_TOKEN":"<REDACTED>"}}'

claude mcp add-json --scope user kubernetes '{"type":"stdio","command":"npx","args":["-y","mcp-server-kubernetes"],"env":{"ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS":"true","KUBECONFIG":"'"$HOME"'/.kube/config.merged"}}'

claude mcp add-json --scope user playwright '{"type":"stdio","command":"npx","args":["@playwright/mcp@latest"],"env":{}}'

claude mcp add-json --scope user vault-mcp-server '{"type":"stdio","command":"docker","args":["run","-i","--rm","-e","VAULT_ADDR","-e","VAULT_TOKEN","hashicorp/vault-mcp-server"],"env":{"VAULT_ADDR":"https://<vault-host>","VAULT_TOKEN":"<REDACTED>"}}'
```

## Prerequisites

Node.js (`npx`) for `gitlab`/`kubernetes`/`playwright`, Docker for the Grafana and Vault servers, and a kubeconfig at `~/.kube/config.merged` for `kubernetes`. To restore on a new machine, run each server's `claude mcp add` command above, substituting real values for `<REDACTED>` and the masked `<...-host>` URLs.
