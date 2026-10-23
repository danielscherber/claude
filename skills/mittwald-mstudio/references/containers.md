# Container Stacks on Mittwald mStudio

## Architecture

- Containers run in **stacks** (docker-compose equivalent)
- Each project has a **default** stack
- Additional stacks can be created via `POST /v2/projects/{projectId}/stacks` (optionally from a container template)
- Each stack has a `prefix` that is prepended to all its service names
- Multiple services and volumes can live in one stack

## Finding Stack IDs

```bash
# Via CLI
mw container list --output json

# Via API
GET /v2/projects/{projectId}/stacks
# The default stack is created with the project; other stacks carry their own description → use its "id"
```

## Creating an Additional Stack

```json
POST /v2/projects/{projectId}/stacks
{
  "description": "monitoring"
}
```

Optionally pass `templateConfig` to create the stack from a container template (see below).

## Container Templates (1-Click Stacks)

Mittwald provides ready-made container templates — including **n8n**, **Directus**, PostgreSQL, MariaDB, Uptime Kuma, Umami, Vaultwarden, Paperless, Open WebUI, Qdrant, Listmonk and many more.

```
GET /v2/container-templates                # list (filters: category, searchTerm, type)
GET /v2/container-templates/{templateId}   # details incl. userInputs (required values)
```

Create a new stack from a template:
```json
POST /v2/projects/{projectId}/stacks
{
  "description": "n8n",
  "templateConfig": {
    "templateId": "n8n",
    "userInputs": [
      { "name": "HOST", "value": "n8n.your-domain.de" },
      { "name": "SMTP_FROM", "value": "info@your-domain.de" }
    ]
  }
}
```

Add a template to an existing stack:
```json
POST /v2/stacks/{stackId}/actions/add-template-component
{
  "templateConfig": { "templateId": "postgresql", "userInputs": [] }
}
```

Missing optional `userInputs` are filled from template defaults. Always check `GET /v2/container-templates/{templateId}` for the required inputs first.

## Full Stack Declaration (PUT)

```json
PUT /v2/stacks/{stackId}

{
  "services": {
    "serviceName": {
      "image": "registry/image:tag",
      "command": ["command", "arg"],
      "entrypoint": ["/entrypoint.sh"],
      "environment": {
        "ENV_VAR": "value"
      },
      "volumes": [
        "volumeName:/container/path",
        "./relative/project/path:/other/path"
      ],
      "ports": ["8080:8080/tcp"],
      "restart": "unless-stopped",
      "x-description": "Human-readable description"
    }
  },
  "volumes": {
    "volumeName": {}
  }
}
```

Supported service fields: `image`, `command`, `entrypoint`, `environment`, `volumes`, `ports`, `restart`, `deploy`, `x-description`. Docker Compose features like `depends_on` or `healthcheck` are **not** part of the API schema.

Deprecated fields (still accepted, avoid in new code): `envs` → `environment`, `restartPolicy` → `restart`, `description` → `x-description`.

⚠️ **PUT is a full replace.** Services omitted from the body are deleted.
Use `PATCH /v2/stacks/{stackId}` to update individual services without touching others.

Volumes removed from the declaration are only detached, not deleted. To delete: `DELETE /v2/stacks/{stackId}/volumes/{volumeId}`.

## Partial Update (PATCH)

```json
PATCH /v2/stacks/{stackId}

{
  "services": {
    "n8n": {
      "environment": {
        "NEW_VAR": "new_value"
      }
    }
  }
}
```

## Automatic Image Updates

Set a cron schedule on the stack (replaces the deprecated `PUT /v2/stacks/{stackId}/update-schedule`):

```json
PATCH /v2/stacks/{stackId}
{
  "x-update-schedule": { "cron": "0 4 * * 1", "timezone": "Europe/Berlin" }
}
```

Set to `null` to remove the schedule.

## Deploy Webhook (CI/CD)

Redeploy a service after a CI build pushes a new image:

```
POST /v2/stacks/{stackId}/services/{serviceId}/webhook-token
→ { "token": "...", "webhookUrl": "https://api.mittwald.de/v2/stacks/.../webhooks/pull-image?token=..." }
```

The token and URL are shown **only once** — store them as a CI secret, never in a repository. Calling the token endpoint again rotates the token.

Trigger from CI:
```bash
curl -fsS "$MITTWALD_PULL_WEBHOOK_URL"
```

## n8n Example Stack (manual)

Prefer the `n8n` container template. Manual equivalent:

```json
{
  "services": {
    "n8n": {
      "image": "n8nio/n8n:latest",
      "environment": {
        "WEBHOOK_URL": "https://n8n.your-domain.de/",
        "N8N_HOST": "n8n.your-domain.de",
        "N8N_PROTOCOL": "https"
      },
      "volumes": [
        "n8n_data:/home/node/.n8n"
      ],
      "ports": ["5678:5678/tcp"],
      "restart": "unless-stopped"
    }
  },
  "volumes": {
    "n8n_data": {}
  }
}
```

Note: current n8n versions use built-in user management — `N8N_BASIC_AUTH_*` variables are no longer supported.

## Private Registries

Pre-defined (no setup needed):
- `index.docker.io`
- `registry.gitlab.com`
- `ghcr.io`

Add a private registry:
```json
POST /v2/projects/{projectId}/registries
{
  "uri": "my-registry.example.com",
  "description": "My private registry",
  "credentials": {
    "username": "user",
    "password": "secret"
  }
}
```

After adding, use images as: `my-registry.example.com/myimage:tag`

Validate before use:
```
POST /v2/actions/validate-container-registry-uri
POST /v2/registries/{registryId}/actions/validate-credentials
```

For existing pre-defined registries with private images:
```json
PATCH /v2/registries/{registryId}
{
  "credentials": { "username": "...", "password": "..." }
}
```

## Connecting a Domain

The ingress targets a container by its **service ID** (from `GET /v2/stacks/{stackId}`) and a port/protocol that the service exposes in `ports`:

```json
POST /v2/ingresses
{
  "hostname": "n8n.your-domain.de",
  "projectId": "<projectId>",
  "paths": [
    {
      "path": "/",
      "target": {
        "container": {
          "id": "<serviceId>",
          "portProtocol": "5678/tcp"
        }
      }
    }
  ]
}
```

Request a TLS certificate: `POST /v2/ingresses/{ingressId}/actions/request-acme-certificate-issuance`

## Container Logs

```bash
# Via API
GET /v2/stacks/{stackId}/services/{serviceId}/logs
GET /v2/stacks/{stackId}/services/{serviceId}/log-analysis   # AI summary + recommendation (Accept-Language: de)

# Via CLI
mw container logs --service-id <id>
mw container logs --service-id <id> --follow   # tail -f equivalent
```

## Lifecycle Operations

```bash
POST /v2/stacks/{stackId}/services/{serviceId}/actions/start
POST /v2/stacks/{stackId}/services/{serviceId}/actions/stop
POST /v2/stacks/{stackId}/services/{serviceId}/actions/restart
POST /v2/stacks/{stackId}/services/{serviceId}/actions/recreate   # recreate container with current config
POST /v2/stacks/{stackId}/services/{serviceId}/actions/pull       # pull latest image + recreate (?skipRecreate=true to only pull)
```

## Volumes

```
GET    /v2/stacks/{stackId}/volumes
GET    /v2/stacks/{stackId}/volumes/{volumeId}
DELETE /v2/stacks/{stackId}/volumes/{volumeId}
```

## Common Issues

| Issue | Cause | Fix |
|---|---|---|
| Service disappears after PUT | PUT replaces entire stack | Always include all services in PUT body, or use PATCH |
| Image pull fails | Private registry not configured | Add registry via POST /registries, validate credentials |
| Container starts but crashes | Check env vars and volume paths | Inspect `logs` or `log-analysis` endpoint |
| Domain returns 502 | Wrong port in ingress | `portProtocol` must match a port in the service's `ports` |
| Container not reachable externally | No ingress defined | Create ingress with POST /v2/ingresses |
| `:latest` image not updated | No automatic pull | Use `actions/pull`, `x-update-schedule` or the deploy webhook |
