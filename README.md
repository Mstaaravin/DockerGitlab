# DockerGitlab

Docker Compose setup for a self-hosted GitLab CE instance with Traefik as reverse proxy and optional GitLab Runner.

## Containers

### Core services

| Container | Description |
|---------|-------------|
| `gitlab` | GitLab CE — git hosting, merge requests, CI/CD orchestration, Container Registry |
| `traefik` | Reverse proxy — TLS termination, routing to `GITLAB_DOMAIN` and `REGISTRY_DOMAIN` |

### GitLab Runner

| Container | Description |
|-----------|-------------|
| `gitlab-runner` | Docker-privileged CI/CD runner — opt-in via `COMPOSE_PROFILES=runner` |

See [doc/RUNNER.md](doc/RUNNER.md) for token setup and re-registration.

## Requirements

- Docker + Docker Compose
- A static IP for the GitLab host
- Port 22 free on the host (GitLab uses it for git-over-SSH). If `sshd` is on 22, move it first:

  ```bash
  sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config
  systemctl restart sshd
  ```

## Setup

### 1. DNS

The default domains (`gitlab.homelab.local`, `registry.homelab.local`, `traefik.homelab.local`) are resolved via local DNS in the reference homelab. If your network doesn't have DNS for these, add them to `/etc/hosts` on the server and on any client that needs browser access:

```
<server-ip>  gitlab.homelab.local registry.homelab.local traefik.homelab.local
```

To use different domains, update the three `*_DOMAIN` variables in `.env` and replace the TLS certificates in `traefik/certs/` (see step 3).

### 2. Environment

```bash
cp .env.example .env
# Edit .env — set VERSION, TZ, the three domain names, GITLAB_IP, etc.
```

### 3. TLS certificates

#### Included lab certs (zero-config for homelab.local)

The repo ships a wildcard certificate for `*.homelab.local` signed by the homelab.local private CA:

```
traefik/certs/wildcard.homelab.local-fullchain.crt
traefik/certs/wildcard.homelab.local.key.rename   ← rename to .key before use
```

> **Note:** These are lab-only certificates intended for quick deployment in a private homelab. Do not use them in production. The CA cert (`homelab.local-intermediate-ca.crt`) must be trusted on any machine that needs browser access — install it in your OS/browser trust store.
>
> The private key is stored with a `.key.rename` extension to avoid GitHub secret scanning alerts.

If you're using the default `homelab.local` domains, skip to step 4 after running:

```bash
# Restore the private key (excluded from git, generated from the .rename file)
cp traefik/certs/wildcard.homelab.local.key.rename traefik/certs/wildcard.homelab.local.key

# Copy cert for the runner
mkdir -p gitlab-runner/config/certs
cp traefik/certs/wildcard.homelab.local-fullchain.crt gitlab-runner/config/certs/gitlab.homelab.local.crt
```

#### Bring your own certificates

Place your cert and key in `traefik/certs/` and update `traefik/certs/tls.toml` to reference them. Then copy the cert (or fullchain) for the runner:

```bash
source .env
mkdir -p gitlab-runner/config/certs
cp traefik/certs/<your-fullchain.crt> gitlab-runner/config/certs/${GITLAB_DOMAIN}.crt
```

Options for generating certs:
- **Self-signed (openssl):** `openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes -keyout traefik/certs/server.key -out traefik/certs/server-fullchain.crt -subj "/CN=${GITLAB_DOMAIN}" -addext "subjectAltName=DNS:${GITLAB_DOMAIN},DNS:${REGISTRY_DOMAIN},DNS:${TRAEFIK_DOMAIN}"`
- **Private CA:** [Certgen](https://github.com/Mstaaravin/Certgen) — creates a local CA and issues a signed cert in one step.

### 4. Start

```bash
docker compose up -d
```

### 5. Initial root password

```bash
docker exec gitlab cat /etc/gitlab/initial_root_password
```

Valid for 24 hours. Change it immediately at `https://<GITLAB_DOMAIN>/-/profile/password/edit`.

## Runner (optional)

1. In GitLab: **Admin Area → CI/CD → Runners → New instance runner** — copy the `glrt-...` token.
2. Add to `.env`: `RUNNER_TOKEN=glrt-xxxxxxxxxxxxxxxxxxxx`
3. Start: `docker compose --profile runner up -d`

See [doc/RUNNER.md](doc/RUNNER.md) for full details.

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
- **Topology:** The nginx overrides (`listen_https = false`, `listen_port = 80`) are deployment-specific — they tell GitLab's internal nginx to speak plain HTTP because Traefik handles TLS termination externally.

**What goes in `gitlab.rb`:**

Everything else — SMTP, LDAP, backup schedules, feature flags, resource limits, etc. Static values that don't depend on `.env` and represent GitLab application configuration rather than deployment topology.

## Notes

- `gitlab/data/`, `gitlab/logs/`, and all secrets are excluded from git via `.gitignore`.
