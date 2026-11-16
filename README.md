# Temporal Auth Server
[![Static Badge for Temporal Code Exchange](https://img.shields.io/badge/Temporal-Code_Exchange_Featured-blue?style=flat-square&logo=temporal&labelColor=141414&color=444CE7)](https://temporal.io/code-exchange/role-based-authentication-for-self-hosted-temporal) 

This repository provides a custom authentication and authorization solution for Temporal, integrating with a custom OIDC provider for user authentication and role-based access control.

## Features

- OIDC Authentication
- Role-based access control for Temporal namespaces
- Environment-based configuration
- Docker Compose overlay for existing Temporal stacks
- Kubernetes / Tilt local development setup

## Prerequisites

- Docker and Docker Compose v2
- Go 1.23 or later
- For Kubernetes development: Docker Desktop with Kubernetes enabled, [Tilt](https://tilt.dev/), and [Helm](https://helm.sh/)

## Getting Started

1. Clone the repository:
```bash
git clone https://github.com/your-repo/temporal-auth-server.git
cd temporal-auth-server
```

2. Copy the environment template and fill in your OIDC credentials:
```bash
cp .env.template .env
```

## Configuration

### Environment Variables

| Variable | Description |
|---|---|
| `TEMPORAL_OIDC_ISSUER_URL` | OIDC provider base URL |
| `TEMPORAL_OIDC_CLIENT_ID` | OAuth client ID |
| `TEMPORAL_OIDC_CLIENT_SECRET` | OAuth client secret |
| `TEMPORAL_AUTH_CALLBACK_URL` | UI SSO callback URL (default: `http://localhost:8080/auth/sso/callback`) |

### Authorization

Access control is managed through group mappings:

- `admin`: Full system access
- `bitovi`: Writer access to `bitovi-project`, reader access to `bitovi-reviews`
- `finance`: Writer access to `finance-reports`, reader access to `finance-audits`

## Docker Compose

`docker-compose.yml` is an **overlay** designed to be merged with an existing Temporal Docker Compose stack. It replaces the stock `temporalio/server` service with the custom RBAC/OIDC server and injects auth configuration into the Temporal UI. All other services — PostgreSQL, OpenSearch, schema setup, namespace init — come from the base stack unchanged.

The server config in `docker/temporal-config.yaml` targets the service names used by [`temporalio/docker-compose`](https://github.com/temporalio/docker-compose): `postgresql` for the database and `opensearch` for visibility.

### Configuration and secrets

Both `docker/temporal-config.yaml` (Compose) and `server/config/development.yaml` (baked into the image, used by k8s) begin with `# enable-template`, which makes Temporal's config loader render them as Go templates (with [sprig](https://masterminds.github.io/sprig/) functions) **at startup**. Connection settings and secrets are read from environment variables, so no credentials are committed or baked into the image:

| Variable | Default | Notes |
|---|---|---|
| `POSTGRES_PWD` | — | **Required**; the server refuses to start without it |
| `POSTGRES_USER` | `temporal` | |
| `POSTGRES_SEEDS` / `DB_PORT` | `postgresql` / `5432` | k8s default host is `postgres-postgresql.postgres.svc.cluster.local` |
| `DBNAME` / `VISIBILITY_DBNAME` | `temporal` / `temporal_visibility` | |
| `SQL_TLS_ENABLED` | `false` | Enable for any non-local database |
| `ES_SCHEME` / `ES_SEEDS` / `ES_PORT` | `http` / `opensearch` / `9200` | Compose only |
| `TEMPORAL_LOG_LEVEL` | `info` | |

How each target supplies them:

- **Local Compose:** copy `.env.template` to `.env` (gitignored) and set `POSTGRES_PWD`. The variable names match the upstream `temporalio/docker-compose` `.env`.
- **Dokploy / other hosts:** set the variables in the platform's environment or secrets UI. Don't commit a `.env`.
- **Kubernetes (Tilt):** the Tiltfile creates the `temporal-db` Secret from `POSTGRES_PWD` in `.env`. The Postgres pod, the Helm schema jobs (`existingSecret`) and the server (`additionalEnv`) all read from it. In production, manage that Secret with External Secrets Operator or Sealed Secrets instead.

### Usage

```bash
docker compose \
  -f /path/to/temporal-server-operations/compose/docker-compose-postgres-opensearch.yml \
  -f docker-compose.yml \
  up --build
```

For Dokploy or other orchestrators that support multiple compose files, configure both files as the stack definition.

### Version Compatibility

The `go.temporal.io/server` version embedded in `server/go.mod` must be compatible with the schema created by the base stack's `temporal-admin-tools` image. If you upgrade the base stack, update the server dependency to match:

```bash
cd server && go get go.temporal.io/server@<version> && go mod tidy
```

### Ports

| Service | Port | Description |
|---|---|---|
| Temporal Frontend (gRPC) | 7233 | Main API endpoint |
| Temporal Frontend (HTTP) | 7243 | HTTP API endpoint |
| Temporal Web UI | 8080 | Web interface |
| PostgreSQL | 5432 | Database (from base stack) |
| pprof | 7936 | Go profiling endpoint |

## Kubernetes / Tilt Development

The Kubernetes setup uses Tilt with Docker Desktop and Helm to run the full stack locally.

### Setup

```bash
# Ensure Docker Desktop Kubernetes is enabled, then:
tilt up
```

Tilt will build the custom server image, apply the Helm chart with `k8s/dev/values-dev.yaml`, and port-forward all services automatically.

### Port Forwards (Tilt)

- PostgreSQL: `localhost:5432`
- Temporal Frontend: `localhost:7233`
- Temporal Web UI: `localhost:8080`

### Making Changes

1. Update the code in `server/`
2. Tilt will automatically rebuild and redeploy
3. Check the Tilt UI for build and deployment status

## Project Structure

```
├── docker/
│   └── temporal-config.yaml   # Server config for Docker Compose (postgres + opensearch)
├── server/
│   ├── config/
│   │   └── development.yaml   # Server config for Kubernetes
│   ├── main.go                # OIDC claim mapper and RBAC authorizer
│   ├── Dockerfile
│   └── go.mod
├── k8s/
│   └── dev/
│       ├── postgres.yaml      # Standalone PostgreSQL manifest
│       └── values-dev.yaml    # Helm values
├── docker-compose.yml         # Compose overlay
├── Tiltfile
└── .env.template
```

## Usage

### Accessing the Web UI

The Temporal Web UI is available at: http://localhost:8080

### Using the API

The Temporal gRPC API is available at: `localhost:7233`

## Troubleshooting

1. **Authentication Failures**
   - Verify `.env` contains correct OIDC credentials
   - Confirm the OIDC provider is reachable from the container network
   - Check server logs for token validation errors

2. **Schema Errors on Startup**
   - The embedded `go.temporal.io/server` version must match the `temporal-admin-tools` version used by the base stack. See [Version Compatibility](#version-compatibility) above.

3. **Build Failures**
   - Ensure Docker is running
   - For Kubernetes: verify the correct context is active (`kubectl config current-context`)

4. **Creating a New Namespace**
   - `tctl --ns <namespace> namespace register`

## Contributing

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request
