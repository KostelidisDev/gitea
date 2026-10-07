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
| `GITEA_NETWORK_SUBNET_V4` / `_V6` | IPv4 and IPv6 ranges of the `gitea` network shared with Traefik (default `10.100.0.0/24`, `fd00:100::/64`). |
| `DATABASE_NETWORK_SUBNET_V4` / `_V6` | IPv4 and IPv6 ranges of the internal `gitea_database` network (default `10.100.1.0/24`, `fd00:100:1::/64`). |
| `GITEA_TAG` / `POSTGRES_TAG` | Pinned image versions. |
| `POSTGRES_DB` / `POSTGRES_USER` | Database name and user. Must match an existing database. |
| `TZ` | Container timezone (default `Europe/Athens`). |
| `*_CPU_LIMIT`, `*_MEM_LIMIT`, `*_MEM_RESERVATION`, `*_PIDS_LIMIT` | Resource limits for `gitea` and `database`. |

## Networks

Both networks are dual-stack (IPv4 and IPv6) with fixed address ranges set in `.env`. Pick ranges that don't overlap other Docker networks, the LAN or a VPN.

> [!NOTE]
> Docker doesn't change the settings of an existing network. After enabling IPv6 or changing a range, recreate the networks: disconnect Traefik (`docker network disconnect gitea <traefik-container>`), run `docker compose down && docker compose up -d`, then reconnect Traefik or restart its stack.

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
| `gitea-data` | Repositories, config (`app.ini`), attachments, avatars |
| `gitea-packages` | Package registry storage (network share) |
| `gitea-database-data` | PostgreSQL data |

The database volume is mounted at `/var/lib/postgresql`, the Postgres 18 image's default layout, so the data lives under `18/docker` inside the volume.

> [!CAUTION]
> - Renaming a volume or changing the database mount path makes Compose use a new, empty volume. Migrate the data first (see below).
> - Changing `POSTGRES_DB` or `POSTGRES_USER` on an initialised volume does not rename the database or user. Gitea will fail to connect.

### Migrating from the old volume names

Earlier deployments used `gitea_data`, `gitea_packages` and `gitea_database_data`, with the database mounted at `/var/lib/postgresql/18`. To move the data into the new volumes:

```sh
docker compose down
docker compose create   # creates the new, empty volumes without starting anything
docker run --rm -v gitea_data:/from:ro -v gitea-data:/to alpine cp -a /from/. /to/
docker run --rm -v gitea_database_data:/from:ro -v gitea-database-data:/to alpine \
  sh -c 'mkdir -p /to/18 && cp -a /from/. /to/18/'
docker compose up -d
```

The old database volume holds `docker/` at its root, so it is copied into `18/` to match the new mount. `gitea-packages` needs no copy: it points to the same network share. Keep the old volumes until you have checked that everything works, then remove them with `docker volume rm`.

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
