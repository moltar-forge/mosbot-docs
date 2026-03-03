---
id: workspace-service
title: Workspace Service
sidebar_label: Workspace Service
sidebar_position: 4
---

The workspace service is a lightweight HTTP REST sidecar **provided by MosBot OS** — it is not part
of the standard OpenClaw distribution. It runs alongside OpenClaw, shares the same workspace
directory or volume, and exposes the workspace filesystem over HTTP so MosBot API can read and write
workspace files, manage configuration, and list agents.

:::info Deployment

You need to run the workspace service container (`ghcr.io/bymosbot/mosbot-workspace-service:latest`)
next to your OpenClaw instance. The image supports both `linux/amd64` and `linux/arm64` platforms.

See [Setting Up OpenClaw](./setup) for Docker and Kubernetes deployment examples. :::

## Recommended mount configuration

For full MosBot functionality (agent discovery + Projects/Skills/Docs file edits), use:

- `WORKSPACE_ROOT=/workspace`
- `WORKSPACE_SUBDIR=.`
- A read-write mount of your OpenClaw home directory (for example `~/.openclaw:/workspace`)

This avoids accidental `/workspace/workspace` nesting and ensures `openclaw.json` is visible to the
workspace service.

## What the workspace service provides

- **File access**: read, create, update, and delete files in agent workspaces
- **Directory listing**: list files and directories recursively
- **Configuration access**: read and write `openclaw.json`
- **Agent discovery**: list agents defined in `openclaw.json`

## Architecture

```text
MosBot API
    │
    │ HTTP (Bearer token auth)
    ▼
OpenClaw Workspace Service (port 8080)
    │
    │ Filesystem access
    ▼
Workspace PVC / directory
(workspace-coo/, workspace-cto/, skills/, shared/, etc.)
```

In Kubernetes, the workspace service runs as a sidecar container in the OpenClaw pod and shares the
same PVC (Persistent Volume Claim). This means it has direct filesystem access to all workspace
directories.

## Authentication

The workspace service uses bearer token authentication. The token is shared between the workspace
service configuration and MosBot API's `OPENCLAW_WORKSPACE_TOKEN` environment variable.

### Generating a token

```bash
WORKSPACE_TOKEN="$(openssl rand -base64 32)"
echo "Save this token: ${WORKSPACE_TOKEN}"
```

Configure the same token in:

1. OpenClaw's workspace service (as `WORKSPACE_SERVICE_TOKEN` in its environment)
2. MosBot API's `.env` (as `OPENCLAW_WORKSPACE_TOKEN`)

## Security model

The workspace service is designed to be an **internal service** — not exposed to the public
internet:

- **Network isolation**: in Kubernetes, deploy as a ClusterIP service (no public ingress)
- **Bearer token auth**: all requests require a valid token
- **Path validation**: the service normalizes paths and rejects directory traversal attempts (e.g.
  `..`)
- **RBAC**: MosBot API enforces its own role-based access control before proxying requests to the
  workspace service

## Workspace directory structure

A typical OpenClaw workspace layout:

```text
/                           ← workspace root
├── workspace-coo/          ← agent workspace (COO agent)
│   ├── memory/             ← agent memory files
│   ├── skills/             ← agent-specific skills
│   └── HEARTBEAT.md        ← heartbeat context file
├── workspace-cto/          ← agent workspace (CTO agent)
│   ├── memory/
│   └── skills/
├── skills/                 ← shared skills (available to all agents)
├── docs/                   ← shared documentation
├── projects/               ← shared project files
└── openclaw.json           ← OpenClaw configuration
```

## API endpoints (via MosBot API)

MosBot API proxies workspace requests through its own authenticated endpoints:

| Endpoint                                 | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| `GET /api/v1/openclaw/workspace/status`  | Check workspace service connectivity     |
| `GET /api/v1/openclaw/workspace/files`   | List files (params: `path`, `recursive`) |
| `GET /api/v1/openclaw/workspace/file`    | Read a file (param: `path`)              |
| `POST /api/v1/openclaw/workspace/file`   | Create a new file                        |
| `PUT /api/v1/openclaw/workspace/file`    | Update an existing file                  |
| `DELETE /api/v1/openclaw/workspace/file` | Delete a file                            |
| `GET /api/v1/openclaw/config`            | Read `openclaw.json`                     |
| `PUT /api/v1/openclaw/config`            | Update `openclaw.json`                   |
| `GET /api/v1/openclaw/agents`            | List agents from `openclaw.json`         |

## Verifying the workspace service

```bash
# Direct check (bypassing MosBot API)
curl -H "Authorization: Bearer <workspace-token>" \
  http://localhost:8080/status

# Via MosBot API (requires MosBot JWT)
curl -H "Authorization: Bearer <mosbot-jwt>" \
  http://localhost:3000/api/v1/openclaw/workspace/status
```

## Troubleshooting

**503 on workspace endpoints** The workspace service is unreachable. Check:

- `OPENCLAW_WORKSPACE_URL` is correct
- The workspace service is running
- Network connectivity between MosBot API and the workspace service

**401 Unauthorized** The token doesn't match. Verify `OPENCLAW_WORKSPACE_TOKEN` in MosBot API's
`.env` matches the token configured in the workspace service.

**Path traversal errors** The path contains `..` or other traversal sequences. Use absolute paths
from the workspace root (e.g. `/workspace-coo/memory/2026-03-01.md`).
