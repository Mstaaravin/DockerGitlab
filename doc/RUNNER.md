# GitLab Runner

This setup includes an optional GitLab Runner that uses a **Docker executor in privileged mode** (Docker-in-Docker). It is disabled by default and activated via a Compose profile.

## Prerequisites

- GitLab CE must be running and accessible.
- The TLS certificate for `GITLAB_DOMAIN` must be placed at:
  ```
  gitlab-runner/config/certs/<GITLAB_DOMAIN>.crt
  ```
  This is the same certificate used by Traefik and GitLab. The runner uses it to verify the GitLab TLS connection, since the certificate is self-signed or issued by a private CA.
- `GITLAB_IP` must be set in `.env` to the static IP of the host running GitLab. The runner container cannot use DNS to resolve `GITLAB_DOMAIN` — it uses `extra_hosts` to inject a host entry directly.

## Getting the registration token

1. Log into GitLab as an administrator.
2. Go to **Admin Area → CI/CD → Runners** (`/admin/runners`).
3. Click **New instance runner**.
4. Configure the runner (tags are optional for an instance runner).
5. Copy the token shown after creation (starts with `glrt-`).
6. Set it in `.env`:
   ```env
   RUNNER_TOKEN=glrt-xxxxxxxxxxxxxxxxxxxx
   ```

## Enabling the runner

The runner service is gated behind the `runner` Compose profile. To include it:

```bash
# via .env (persists across restarts):
COMPOSE_PROFILES=runner

# or inline:
docker compose --profile runner up -d
```

To disable it, remove `runner` from `COMPOSE_PROFILES` and stop the container:

```bash
docker compose stop gitlab-runner
```

## Auto-registration on first start

On startup, the entrypoint checks whether `config.toml` exists and is non-empty. If not, it runs `gitlab-runner register` with the values from `.env` and generates the file automatically.

```
gitlab-runner/config/config.toml   ← auto-generated, not tracked in git
```

Subsequent restarts skip registration and go directly to `gitlab-runner run`.

The registered runner uses:

| Setting | Value |
|---------|-------|
| Executor | `docker` |
| Base image | `RUNNER_DOCKER_IMAGE` (e.g. `docker:27`) |
| Privileged mode | yes (required for Docker-in-Docker) |
| Extra volumes | `/certs/client` (DinD TLS), `/cache` |
| TLS CA file | `certs/<GITLAB_DOMAIN>.crt` |

## Docker-in-Docker (DinD)

Privileged mode and the `/certs/client` volume are required for DinD pipelines. Jobs that build Docker images should use the `docker:dind` service:

```yaml
# .gitlab-ci.yml example
build:
  image: docker:27
  services:
    - docker:dind
  variables:
    DOCKER_TLS_CERTDIR: /certs
  script:
    - docker build -t myimage .
```

## Re-registering the runner

To force re-registration (e.g. after a token rotation):

1. Delete or empty `gitlab-runner/config/config.toml` on the server.
2. Update `RUNNER_TOKEN` in `.env`.
3. Restart the container:
   ```bash
   docker compose --profile runner restart gitlab-runner
   ```
