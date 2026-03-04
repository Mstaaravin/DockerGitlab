# DockerGitlab

Docker Compose setup for a self-hosted GitLab CE instance with Traefik as reverse proxy and optional GitLab Runner.

## Services

### Core services

| Service | Description |
|---------|-------------|
| `gitlab` | GitLab CE — git hosting, merge requests, CI/CD orchestration, Container Registry |
| `traefik` | Reverse proxy — TLS termination, routing to `GITLAB_DOMAIN` and `REGISTRY_DOMAIN` |

### GitLab Runner

| | |
|---|---|
| Profile | opt-in via `COMPOSE_PROFILES=runner` |
| Executor | Docker privileged (Docker-in-Docker) |
| Registration | Automatic on first start using `RUNNER_TOKEN` |
| Config file | `gitlab-runner/config/config.toml` — auto-generated, not tracked in git |

See [doc/RUNNER.md](doc/RUNNER.md) for token setup, TLS requirements, and re-registration.

## Requirements

- Docker + Docker Compose
- TLS certificates for your domains (self-signed or CA-issued)
- A static IP for the GitLab host (required for runner DNS resolution)

## Setup

1. Copy `.env.example` to `.env` and fill in the values:

   ```env
   VERSION=18.x.x-ce.0          # GitLab CE image tag
   TZ=Your/Timezone
   GITLAB_DOMAIN=gitlab.example.com
   REGISTRY_DOMAIN=registry.example.com
   TRAEFIK_DOMAIN=traefik.example.com
   GITLAB_IP=<server-ip>
   RUNNER_DOCKER_IMAGE=docker:27
   COMPOSE_PROFILES=             # Set to "runner" to include GitLab Runner
   RUNNER_TOKEN=<token>          # Admin > CI/CD > Runners > New instance runner
   ```

2. Place TLS certificates:
   - `traefik/certs/<GITLAB_DOMAIN>.crt` and `.key`
   - `gitlab/config/ssl/<GITLAB_DOMAIN>.crt` and `.key`

3. Configure `traefik/certs/*.toml` and `gitlab/config/ssl/*.toml` with your domain names.

4. Start:

   ```bash
   docker compose up -d
   # With runner:
   docker compose --profile runner up -d
   ```

## Configuration split: `GITLAB_OMNIBUS_CONFIG` vs `gitlab.rb`

GitLab CE reads configuration from two sources, applied in this order:

1. `GITLAB_OMNIBUS_CONFIG` (environment variable in `compose.yml`) — prepended first.
2. `gitlab/config/gitlab.rb` (bind-mounted file) — appended after, so it takes precedence for any duplicate keys.

**What goes in `GITLAB_OMNIBUS_CONFIG`:**

```ruby
external_url 'https://${GITLAB_DOMAIN}'
registry_external_url 'https://${REGISTRY_DOMAIN}'
nginx['listen_https'] = false
nginx['listen_port'] = 80
gitlab_pages['enable'] = true
```

These settings live here for two reasons:

- **Variable interpolation:** Docker Compose substitutes `${GITLAB_DOMAIN}` and `${REGISTRY_DOMAIN}` from `.env` before the container starts. `gitlab.rb` is a static file committed to git and has no access to `.env` variables.
- **Topology:** The nginx overrides (`listen_https = false`, `listen_port = 80`) are deployment-specific — they tell GitLab's internal nginx to speak plain HTTP because Traefik handles TLS termination externally. They belong with the other deployment-time settings, not in the general application config.

**What goes in `gitlab.rb`:**

Everything else — SMTP, LDAP, backup schedules, feature flags, resource limits, etc. These are static values that don't depend on `.env` and represent the GitLab application configuration rather than the deployment topology.

## Notes

- `gitlab/data/`, `gitlab/logs/`, and all secrets are excluded from git via `.gitignore`.
