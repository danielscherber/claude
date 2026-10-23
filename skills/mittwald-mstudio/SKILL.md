---
name: mittwald-mstudio
description: >
  Deep knowledge of Mittwald mStudio v2 — CLI (mw), REST API, Mittwald MCP server, Docker
  container stacks and container templates (n8n, Directus, etc.), and App Marketplace setups
  (WordPress, TYPO3, etc.). Use this skill whenever the user asks
  about anything on Mittwald / mStudio: setting up apps, deploying containers, automating
  via the API or CLI, configuring projects/spaces/organizations, debugging deployments, or
  writing shell/automation scripts that interact with mStudio. Also trigger for questions
  about mw CLI commands, API tokens, container stacks, ingress/domain routing, database
  provisioning, or environment variables on Mittwald. If the user mentions "Mittwald",
  "mStudio", "mw CLI", "Space-Server", or "Mittwald project", always consult this skill.
  This skill instructs Claude to always search developer.mittwald.de and github.com/mittwald
  for current solutions before answering — use it any time a Mittwald problem or setup task
  needs a definitive, up-to-date answer.
---

# Mittwald mStudio Skill

This skill covers the Mittwald mStudio v2 platform. It focuses on:

- **mw CLI** — authentication, project/app/container management
- **REST API v2** — `https://api.mittwald.de/v2/` — for automation and scripting
- **Docker container stacks** — deployment, volumes, env vars, ingress, logs
- **App Marketplace** — WordPress, TYPO3, Contao, Shopware and other managed apps
- **Container templates** — 1-click stacks for n8n, Directus, PostgreSQL, Uptime Kuma, etc.
- **Mittwald MCP server** — direct tool access from Claude, when connected
- **Project & organization management** — IDs, context, resource scoping

For detailed reference beyond what's in this file, read the appropriate reference file:
- `references/cli.md` — CLI commands, flags, context
- `references/api.md` — API concepts, authentication, key endpoints
- `references/containers.md` — Docker stack setup, container templates, API payload patterns
- `references/apps.md` — Marketplace apps, WordPress specifics, system software

---

## Authoritative Sources — Always Search First

**Before answering any Mittwald question, always search these sources.** They contain the most current documentation, changelogs, and community-contributed examples. The skill's reference files are a starting point, not a substitute for current docs.

| Source | What to find there |
|---|---|
| `https://developer.mittwald.de/de/` | Official developer portal: API reference, CLI docs, how-tos, changelogs |
| `https://github.com/mittwald/developer-portal` | Docs source — issues often contain workarounds, open bugs, undocumented behavior |
| `https://github.com/mittwald` | All official Mittwald repos: CLI source, API clients (Go, JS), extensions, examples |

### Search strategy

1. **Use `WebFetch`** to retrieve specific documentation pages directly when you know the URL (e.g. a specific how-to or API endpoint page).
2. **Use `WebSearch`** with queries like:
   - `site:developer.mittwald.de <topic>`
   - `site:github.com/mittwald <topic>`
   - `mittwald mStudio <topic> <year>` for recent issues or changelogs
3. **Check GitHub Issues** on `https://github.com/mittwald/developer-portal/issues` and `https://github.com/mittwald/cli/issues` for known bugs, workarounds, and recent changes that may not be in the main docs yet.

### Key documentation URLs

```
# Developer portal (DE)
https://developer.mittwald.de/de/docs/v2/api/intro/
https://developer.mittwald.de/de/docs/v2/cli/
https://developer.mittwald.de/de/docs/v2/category/how-tos/
https://developer.mittwald.de/changelog/

# API reference
https://developer.mittwald.de/de/docs/v2/reference/

# GitHub
https://github.com/mittwald/cli
https://github.com/mittwald/developer-portal
https://github.com/mittwald/api-client-js
https://github.com/mittwald/api-client-go
```

**OpenAPI spec (always current, public):** `https://api.mittwald.de/v2/openapi.json` — the fastest way to verify an endpoint, payload schema or deprecation before answering.

---

## Mittwald MCP Server — Prefer When Connected

If tools named `mittwald_*` are available in the session, use them for **actions** instead of CLI or raw API calls. They cover:

| Area | Example tools |
|---|---|
| Projects, servers, orgs | `mittwald_project_list`, `mittwald_server_list`, `mittwald_org_membership_list` |
| Apps | `mittwald_app_list`, `mittwald_app_upgrade`, `mittwald_app_versions` |
| Containers & stacks | `mittwald_stack_deploy`, `mittwald_stack_ps`, `mittwald_container_logs`, `mittwald_container_restart` |
| Databases | `mittwald_database_mysql_create`, `mittwald_database_mysql_dump`, `mittwald_database_redis_create` |
| Domains & DNS | `mittwald_domain_dnszone_update`, `mittwald_domain_virtualhost_create`, `mittwald_certificate_request` |
| Backups & cronjobs | `mittwald_backup_create`, `mittwald_backup_schedule_create`, `mittwald_cronjob_create` |
| Mail | `mittwald_mail_address_create`, `mittwald_mail_deliverybox_create` |
| Access | `mittwald_ssh_user_create`, `mittwald_user_api_token_create` |

Rules:
- Read first (`*_list`, `*_get`) to resolve IDs — never guess IDs.
- Destructive tools (`*_delete`, `*_uninstall`, `stack_deploy` replacing services, `dnszone_update`) need explicit user confirmation.
- If the MCP server is not connected, fall back to the `mw` CLI or the REST API below.

---

## Core Concepts

### Resource Hierarchy

```
Organization (--org-id)
  └── Server / Space (--server-id)
        └── Project (--project-id)
              ├── App Installations (--installation-id)
              ├── Container Stacks
              ├── Databases (MySQL, Redis)
              └── Domains / Ingresses
```

Most CLI commands require a `--project-id` or `--installation-id`. Set a persistent context to avoid repeating flags.

### Authentication

**API Token** (required for both CLI and direct API calls):
```bash
# Interactive login
mw login token

# Non-interactive / CI / n8n workflows
export MITTWALD_API_TOKEN=<token>
mw login status
```

Token permissions: `api_read`, `api_write` — set during creation in mStudio UI under *Profil → API-Tokens* or via API:
```
POST /v2/users/self/api-tokens
{ "description": "...", "roles": ["api_read", "api_write"] }
```

---

## CLI Quick Reference

### Installation (macOS)
```bash
brew install mittwald/tap/cli
```

### Persistent Context
```bash
mw context set --project-id <id>     # scope all commands to a project
mw context set --installation-id <id>
mw context get                        # show current context
```

### Key Command Groups

| Command group | Purpose |
|---|---|
| `mw project` | List, create, manage projects |
| `mw app` | Manage app installations |
| `mw container` | Container stack management |
| `mw database` | MySQL / Redis provisioning |
| `mw domain` | Domain & DNS management |
| `mw user ssh-key` | SSH key management |
| `mw context` | Persistent project/server context |

### Common Flags
- `--project-id` / `-p` — target project
- `--output json` — machine-readable output (useful for scripting)
- `--quiet` / `-q` — suppress interactive prompts (for CI/automation)
- `--ssh-identity-file` — specify SSH key for `mw app ssh`, `mw project ssh`

---

## Docker Container Stacks

Containers on Mittwald run as **container stacks** (analogous to docker-compose). Each project has a **default stack**; additional stacks can be created via `POST /v2/projects/{projectId}/stacks`.

### Container templates (check first!)

Before writing a stack by hand, check whether a template exists:
```
GET /v2/container-templates?searchTerm=<name>
GET /v2/container-templates/{templateId}     # required userInputs
```
Available templates include n8n, Directus, PostgreSQL, MariaDB, Uptime Kuma, Umami, Vaultwarden, Paperless, Open WebUI, Qdrant, Listmonk, Mailpit and more. See `references/containers.md`.

### API: Deploy/Update a container

```
PUT /v2/stacks/{stackId}
```

**⚠️ Idempotent replace**: This replaces the entire stack — omitted services are deleted. Always include all existing services when adding a new one. Use `PATCH /v2/stacks/{stackId}` to update selectively.

Minimal payload:
```json
{
  "services": {
    "n8n": {
      "image": "n8nio/n8n:latest",
      "environment": {
        "N8N_HOST": "n8n.your-domain.de",
        "N8N_PROTOCOL": "https",
        "WEBHOOK_URL": "https://n8n.your-domain.de/"
      },
      "volumes": ["n8n_data:/home/node/.n8n"],
      "ports": ["5678:5678/tcp"],
      "restart": "unless-stopped"
    }
  },
  "volumes": {
    "n8n_data": {}
  }
}
```

### Pre-defined registries (no manual setup needed)
- `index.docker.io` (Docker Hub)
- `registry.gitlab.com`
- `ghcr.io`

For private registries: `POST /v2/projects/{projectId}/registries`

### Container lifecycle (API)
```
POST /v2/stacks/{stackId}/services/{serviceId}/actions/start
POST /v2/stacks/{stackId}/services/{serviceId}/actions/stop
POST /v2/stacks/{stackId}/services/{serviceId}/actions/restart
POST /v2/stacks/{stackId}/services/{serviceId}/actions/recreate
POST /v2/stacks/{stackId}/services/{serviceId}/actions/pull        # pull latest image + recreate
GET  /v2/stacks/{stackId}/services/{serviceId}/logs
GET  /v2/stacks/{stackId}/services/{serviceId}/log-analysis        # AI log summary
```

### Automatic updates & CI deploys
- Image update schedule: `PATCH /v2/stacks/{stackId}` with `"x-update-schedule": { "cron": "0 4 * * 1", "timezone": "Europe/Berlin" }`
- Deploy webhook for CI: `POST /v2/stacks/{stackId}/services/{serviceId}/webhook-token` → returns a one-time `webhookUrl`; store it as a CI secret, call it after pushing a new image.

### Connecting a domain to a container
Use ingress: `POST /v2/ingresses` — target `{ "container": { "id": "<serviceId>", "portProtocol": "5678/tcp" } }`. The port must be exposed in the service's `ports`.

---

## App Marketplace (Managed Apps)

Apps like WordPress, TYPO3, Matomo, Nextcloud, Shopware, plus generic Node.js / Python / PHP / Static Files runtimes, are managed via:
```
POST /v2/projects/{projectId}/appinstallations
GET  /v2/projects/{projectId}/appinstallations
```

Or via CLI:
```bash
mw app list                          # available apps
mw app install wordpress             # guided install
mw app get --installation-id <id>   # details of installed app
mw app ssh                          # SSH into app context
mw app download                     # download files via SCP/rsync
```

### n8n on Mittwald
n8n is **not** a marketplace app. Recommended approach:
- Use the **`n8n` container template** (`POST /v2/projects/{projectId}/stacks` with `templateConfig.templateId: "n8n"`, required inputs `HOST` and `SMTP_FROM`)
- Manual alternative: Docker Hub image `n8nio/n8n`, named volume on `/home/node/.n8n`, port `5678`
- Set `MITTWALD_API_TOKEN` as env var in the container stack if n8n needs to call the Mittwald API — never hard-code the token in workflow JSON

### WordPress on Mittwald
WordPress is available as a **managed marketplace app**:
```bash
mw app install wordpress
```
- PHP version can be changed via API: `PATCH /v2/appinstallations/{id}/` or through the mStudio UI
- Redis for object cache: provision via `POST /v2/projects/{projectId}/redis-databases` and configure in wp-config.php

---

## REST API Patterns

**Base URL**: `https://api.mittwald.de/v2/`

**Auth header**: `Authorization: Bearer <MITTWALD_API_TOKEN>`

### Get project list
```
GET /v2/projects?customerId=<org-id>
```

### Create a MySQL database
```
POST /v2/projects/{projectId}/mysql-databases
{
  "description": "mydb",
  "database": "mydb",
  "version": "8.0",
  "characterSettings": { "characterSet": "utf8mb4", "collate": "utf8mb4_unicode_ci" }
}
```

### Create a Redis database
```
POST /v2/projects/{projectId}/redis-databases
{
  "description": "myredis",
  "version": "7.0"
}
```

### Change PHP / system software version
```
PATCH /v2/appinstallations/{installationId}
{
  "systemSoftware": {
    "<php-software-id>": { "systemSoftwareVersion": "<version-id>" }
  }
}
```
→ See `references/api.md` for how to look up software and version IDs.

---

## Scripting & Automation Patterns

### Using mw CLI in shell scripts / Hazel / n8n Execute Command node
```bash
#!/bin/bash
export MITTWALD_API_TOKEN="$(cat ~/.config/mw/token)"

# Get project list as JSON
mw project list --output json | jq '.[].id'

# Check container status
mw container list --project-id <id> --output json
```

### Using the API from n8n (HTTP Request node)
- Method: `GET` / `POST` / `PUT` / `PATCH`
- URL: `https://api.mittwald.de/v2/...`
- Auth: Header `Authorization: Bearer {{ $env.MITTWALD_API_TOKEN }}`

### Getting IDs you need
```bash
mw project list --output json        # → project IDs
mw app list --output json            # → installation IDs
mw container list --output json      # → stack + service IDs
mw database list --output json       # → database IDs
```

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `too many authentication failures` | Specify `--ssh-identity-file ~/.ssh/mstudio-cli` or set `MITTWALD_SSH_IDENTITY_FILE` |
| Container won't start | Check logs: `GET /v2/stacks/{stackId}/services/{serviceId}/logs` or `mw container logs` |
| PUT stack wipes services | PUT is a full replace — include ALL services in every PUT call; use PATCH for selective updates |
| App install fails | Verify project has container/app feature enabled (Space-Server required for containers) |
| API 403 | Token missing `api_write` role — regenerate with correct permissions |
| Domain not routing to container | Check ingress definition — `portProtocol` must match a port exposed in the service's `ports` |
| Image `:latest` never updates | Use `actions/pull`, an `x-update-schedule` or the deploy webhook |
| Unsure whether an endpoint/field still exists | Check `https://api.mittwald.de/v2/openapi.json` — deprecated fields are marked there |

---

## Reference Files

Load these when you need deeper detail:

- `references/cli.md` — Full CLI command reference, flags, context system
- `references/api.md` — API authentication, pagination, error codes, key endpoint list
- `references/containers.md` — Full container stack spec, container templates, ingress, update schedules, deploy webhooks
- `references/apps.md` — Marketplace apps, WordPress specifics, system software versioning
