# 🚀 Deploy Laravel Blue‑Green

A [GitHub Action](https://github.com/features/actions) that deploys a Laravel application using a **Blue‑Green deployment strategy** over SSH.  
It manages Docker containers, Caddy reverse proxy, environment files, and health checks – all with zero-downtime traffic switching.

![GitHub release (latest by date)](https://img.shields.io/github/v/release/bugfix666/deploy-laravel-blue-green?label=version)

## How it works

1. **Prepares** `.env` files for both `blue` and `green` environments on the target server, injecting secrets, generating `APP_KEY` / `JWT_SECRET`, and setting up optional credentials for Horizon and Telescope.
2. **Determines** the currently active environment by inspecting the Caddy configuration.
3. **Launches** a new Docker container (the *inactive* color) with the new image, mounts the prepared `.env`, and runs migrations & optimization.
4. **Validates** the new container via a health‑check endpoint.
5. **Switches** traffic in Caddy from the old port to the new port.
6. **Stops** the old container (optional).

All steps happen over a single SSH connection, requiring only `docker`, `make`, `curl`, and `git` on the server.

## Inputs

| Name | Description | Required | Default |
|---|---|---|---|
| **SSH connection** | | | |
| `ssh_host` | SSH server hostname or IP | **yes** | |
| `ssh_user` | SSH username | **yes** | |
| `ssh_key` | SSH private key | **yes** | |
| `ssh_port` | SSH port | no | `22` |
| **Container registry** | | | |
| `registry` | Container registry (e.g. `ghcr.io`) | no | `ghcr.io` |
| `registry_username` | Registry username | **yes** | |
| `registry_password` | Registry password or token | **yes** | |
| `image_name` | Full Docker image name with tag (e.g. `ghcr.io/owner/repo:latest`) | **yes** | |
| **Repository with configs** | | | |
| `repo_ssh_url` | Git SSH URL of the repository containing deployment configs (`Makefile`, `docker-compose.prod.yml`, `.env.example`) | **yes** | |
| `deploy_branch` | Branch from which to clone the configs | no | `deploy` |
| **Server paths** | | | |
| `base_dir` | Base directory on the server (where environment folders are stored) | no | `/www` |
| `health_path` | Health check endpoint (must return HTTP 200) | no | `/health` |
| **Database** | | | |
| `db_host` | Database host | no | |
| `db_port` | Database port | no | |
| `db_username` | Database username | no | |
| `db_password` | Database password | no | |
| **Redis** | | | |
| `redis_host` | Redis host | no | |
| `redis_port` | Redis port | no | |
| `redis_password` | Redis password | no | |
| **Key generation** | | | |
| `generate_app_key` | Auto‑generate `APP_KEY` and `JWT_SECRET` when they are missing | no | `true` |
| `app_key_prefix` | Add `base64:` prefix to `APP_KEY` (`true` = Laravel default, `false` = plain hex) | no | `true` |
| **Application settings** | | | |
| `app_env` | Laravel environment (`production`, `local`, `develop`, etc.) | no | `production` |
| `app_debug` | Enable/disable debug mode (`true`/`false`) | no | `false` |
| **Logging** | | | |
| `log_channel` | Laravel log channel (e.g. `stack`, `daily`, `single`) | no | `stack` |
| `log_level` | Minimum log level (`debug`, `info`, `error`, …) | no | `error` |
| **Horizon auth** | | | |
| `horizon_admin_user` | Username for Horizon dashboard | no | `horizon` |
| `horizon_admin_password` | Password for Horizon dashboard | no | `horizon` |
| **Telescope auth** | | | |
| `telescope_admin_user` | Username for Telescope dashboard | no | `telescope` |
| `telescope_admin_password` | Password for Telescope dashboard | no | `telescope` |

## Usage

### Minimal example (in your `.github/workflows/deploy.yml`)

```yaml
name: Deploy

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy with Blue‑Green
        uses: bugfix666/deploy-laravel-blue-green@v1.0.2
        with:
          ssh_host: ${{ secrets.SSH_HOST }}
          ssh_user: ${{ secrets.SSH_USER }}
          ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
          registry_username: ${{ github.actor }}
          registry_password: ${{ secrets.GHCR_TOKEN }}
          image_name: ghcr.io/${{ github.repository }}:latest
          repo_ssh_url: git@github.com:${{ github.repository }}.git
          db_host: ${{ secrets.DB_HOST }}
          db_port: ${{ secrets.DB_PORT }}
          db_username: ${{ secrets.DB_USERNAME }}
          db_password: ${{ secrets.DB_PASSWORD }}
          redis_host: ${{ secrets.REDIS_HOST }}
          redis_port: ${{ secrets.REDIS_PORT }}
          redis_password: ${{ secrets.REDIS_PASSWORD }}
          # Optional settings (shown with defaults)
          app_env: production
          app_debug: false
          log_channel: stack
          log_level: error
          horizon_admin_user: ${{ secrets.HORIZON_USER }}
          horizon_admin_password: ${{ secrets.HORIZON_PASSWORD }}
          telescope_admin_user: ${{ secrets.TELESCOPE_USER }}
          telescope_admin_password: ${{ secrets.TELESCOPE_PASSWORD }}
```

### Server requirements

The target server must have installed:

- **Docker** and Docker Compose (standalone or plugin)
- **Caddy** web server (any modern version)
- **make**, **curl**, **git**, **openssl**

Your repository must contain (at the branch specified by `deploy_branch`):

- `Makefile` with the following targets:
  - `deploy` – pulls image, runs `docker compose up`, copies `.env`, runs migrations, clears caches.
  - `health-check` – validates the new environment (can be a simple curl command).
  - `switch-to-blue` / `switch-to-green` – updates Caddy configuration and reloads Caddy.
  - `down` – stops containers for a given environment.
- `docker-compose.prod.yml` – production Compose file that accepts `${ENV}`, `${PORT}`, `${IMAGE}` variables.
- `.env.example` – template for Laravel environment variables.

To quickly bootstrap a compatible project, copy the example files from this repository's [`example/`](example/) folder into your Laravel project and adjust them to your needs.

## How traffic switching works

The action maintains a Caddy configuration file (`/www/Caddyfile` by default) that proxies requests to either `localhost:8081` (blue) or `localhost:8082` (green). After a successful health check, it runs the corresponding `make switch-to-*` target which rewrites the file and reloads Caddy. No downtime occurs because Caddy gracefully drains old connections.

## Security

- The SSH private key is never written to disk – it is passed directly to the `appleboy/ssh-action`.
- All secrets are transmitted through GitHub Actions secrets and never appear in logs.
- The `.env` files are stored on the server with restricted permissions (`www-data` owner, `600` mode).

## Contributing

Contributions are welcome! Please open an issue or a pull request.  
Major changes should be discussed first. The action is versioned via Git tags – see [GitHub Actions versioning](https://docs.github.com/en/actions/creating-actions/about-custom-actions#versioning-your-action).
```

Теперь README полностью отражает все возможности, описанные в `action.yml`, включая управление учётными данными Horizon/Telescope, настройки логирования и окружения.