# Gitea

Self-hosted [Gitea](https://gitea.com) with PostgreSQL, run with Docker Compose behind Traefik.

- **Gitea** serves the web UI, API and package registry through Traefik on `https://${DOMAIN}`. Git over SSH is published directly on the host.
- **PostgreSQL** sits on an internal network that only Gitea can reach.
- **Packages** are stored on a network share (NFS or CIFS) mounted as a Docker volume.

## Requirements

- Docker Engine with Compose v2
- A running Traefik instance that:
  - has a `websecure` entrypoint with a default TLS certificate covering `${DOMAIN}`
  - is attached to the `gitea` Docker network so it can reach the container on port 3000
- An NFS or CIFS share for package storage, reachable from the Docker host

## Setup

```sh
cp .env.example .env
# edit .env: set DOMAIN, POSTGRES_PASSWORD and the PACKAGES_MOUNT_* values
docker compose up -d
```

Then open `https://${DOMAIN}` and finish the Gitea install wizard. The server and database settings are already filled in from the environment.

## Configuration

All settings live in `.env`. See [`.env.example`](.env.example) for the full list.

| Variable | Purpose |
| --- | --- |
| `DOMAIN` | Public hostname. Used for the Traefik router, `ROOT_URL` and SSH clone URLs. **Required.** |
| `POSTGRES_PASSWORD` | Database password, passed to both containers as a Docker secret. **Required.** |
| `GITEA_SSH_PORT` | Host port for Git over SSH. Must not clash with the host's own sshd. **Required.** |
| `GITEA_SSH_BIND` | Host interface for the SSH port (default `0.0.0.0`). Set to a VPN IP to keep SSH private. |
| `PACKAGES_MOUNT_TYPE` / `_O` / `_DEVICE` | Network share options for the packages volume. **Required.** |
| `IPV4_ALLOWLIST` / `IPV6_ALLOWLIST` | Source ranges allowed through Traefik (web UI, API, registry). |
| `GITEA_TAG` / `POSTGRES_TAG` | Pinned image versions. |
| `POSTGRES_DB` / `POSTGRES_USER` | Database name and user. Must match an existing database. |
| `TZ` | Container timezone (default `Europe/Athens`). |
| `*_CPU_LIMIT`, `*_MEM_LIMIT`, `*_MEM_RESERVATION`, `*_PIDS_LIMIT` | Resource limits for `gitea` and `database`. |

## Access

| Path | Route | Restricted by |
| --- | --- | --- |
| Web UI, API, package registry | Traefik → `gitea:3000` | `IPV4_ALLOWLIST` / `IPV6_ALLOWLIST` |
| Git over SSH | Host `${GITEA_SSH_BIND}:${GITEA_SSH_PORT}` → `gitea:22` | Bind address only |

Clone URLs over SSH look like:

```
ssh://git@${DOMAIN}:${GITEA_SSH_PORT}/owner/repo.git
```

> [!WARNING]
> The SSH port bypasses Traefik and its allowlist. Docker-published ports also bypass `ufw`. To restrict SSH, bind it to a private interface with `GITEA_SSH_BIND` or add rules to the `DOCKER-USER` chain.

Traefik is configured to allow encoded `/` (`%2F`) and `%` (`%25`) in paths, which npm scoped packages and refs containing `%` need.

## Data

| Volume | Contents |
| --- | --- |
| `gitea_data` | Repositories, config (`app.ini`), attachments, avatars |
| `gitea_packages` | Package registry storage (network share) |
| `gitea_database_data` | PostgreSQL data |

The volume names are set explicitly so they match volumes from earlier deployments.

> [!CAUTION]
> - The database volume is mounted at `/var/lib/postgresql/18` because the existing data lives under `18/docker`. Changing the mount path makes Postgres start with an empty database.
> - Changing `POSTGRES_DB` or `POSTGRES_USER` on an initialised volume does not rename the database or user. Gitea will fail to connect.

## Operations

```sh
docker compose ps                  # status and health
docker compose logs -f gitea       # follow Gitea logs
docker compose pull && docker compose up -d   # upgrade after bumping *_TAG
```

Back up the database:

```sh
docker compose exec database sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' > gitea-$(date +%F).sql
```

Back up Gitea data (repositories, config, attachments):

```sh
docker compose exec -u git gitea gitea dump -c /data/gitea/conf/app.ini -f /tmp/gitea-dump.zip
docker compose cp gitea:/tmp/gitea-dump.zip .
```

Read the Gitea release notes before upgrading a major version. When upgrading PostgreSQL across major versions, migrate the data with `pg_dump`/`pg_restore` or `pg_upgrade` first.
