# nextcloud-stack

Docker Compose stack for running **Nextcloud** behind **Traefik** with PostgreSQL, Redis, Nginx and a dedicated cron container.

The repository is intended as a compact self-hosted infrastructure example: separate application, database, cache and web frontend containers; persistent Docker volumes; reverse proxy integration; and basic security headers.

## Stack

- **Nextcloud FPM** — application container
- **PostgreSQL** — database backend
- **Redis** — cache and file locking backend
- **Nginx** — web frontend for PHP-FPM
- **Traefik** — external reverse proxy and TLS entrypoint
- **Nextcloud cron** — background jobs runner

## Architecture

```text
Internet
   |
Traefik
   |
nextcloud-nginx
   |
nextcloud-app (PHP-FPM)
   |
PostgreSQL / Redis
```

The stack uses two Docker networks:

- `nextcloud_internal` — private internal network for Nextcloud, PostgreSQL and Redis
- `proxy` — external Docker network used by Traefik

## Repository files

```text
docker-compose.yml   # Main Compose stack
nginx.conf           # Nginx configuration for Nextcloud FPM
uploads.conf         # Upload and FastCGI buffer settings
README.md            # Project documentation
```

## Requirements

- Linux server with Docker and Docker Compose plugin
- Existing Traefik reverse proxy
- Existing external Docker network named `proxy`
- DNS record pointing the Nextcloud domain to the server
- Environment variables stored outside Git

## Environment variables

Create a `.env` file near `docker-compose.yml`:

```env
NEXTCLOUD_DOMAIN=cloud.example.com
NEXTCLOUD_DB_PASSWORD=change-me
REDIS_PASSWORD=change-me
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=change-me
```

Do not commit `.env` to Git.

## Prepare Traefik network

```bash
docker network create proxy
```

Skip this step if the `proxy` network already exists.

## Start

```bash
docker compose up -d
```

Check containers:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f nextcloud-app
docker compose logs -f nextcloud-nginx
```

## Services

### PostgreSQL

Stores Nextcloud metadata and application state.

Persistent volume:

```text
nextcloud_db
```

### Redis

Used for cache and file locking.

### Nextcloud app

Runs the official `nextcloud:fpm` image.

Persistent volumes:

```text
nextcloud_data
nextcloud_apps
nextcloud_config
nextcloud_themes
```

### Nginx

Serves static files and proxies PHP requests to the FPM container.

### Cron

Runs Nextcloud background jobs using `/cron.sh`.

## Security notes

- Keep `.env` out of Git.
- Use strong passwords for PostgreSQL, Redis and the Nextcloud admin user.
- Do not expose PostgreSQL or Redis directly to the internet.
- Only the Nginx container should be attached to the Traefik-facing `proxy` network.
- Keep backups outside the application server or sync them to external storage.
- Avoid exposing the Docker socket to application containers.

## Backup

Minimal backup set:

```text
nextcloud_db
nextcloud_data
nextcloud_apps
nextcloud_config
nextcloud_themes
```

Example volume backup approach:

```bash
docker compose down
sudo tar -czf nextcloud-backup-$(date +%Y%m%d).tar.gz /var/lib/docker/volumes/nextcloud-stack_*
docker compose up -d
```

For production, prefer a consistent PostgreSQL dump plus file backup.

Example PostgreSQL dump:

```bash
docker exec nextcloud-db pg_dump -U nextcloud nextcloud > nextcloud-db-$(date +%Y%m%d).sql
```

## Restore

```bash
docker compose down
# restore database and volumes
docker compose up -d
```

After restore, check:

```bash
docker compose logs -f nextcloud-app
docker compose logs -f nextcloud-nginx
```

## Upgrade

Pull newer images and recreate containers:

```bash
docker compose pull
docker compose up -d
```

Check logs after upgrade:

```bash
docker compose logs -f nextcloud-app
```

## Useful commands

```bash
docker compose ps
docker compose logs -f
docker compose pull
docker compose up -d
docker compose down
```

## Current limitations

- Images are not pinned to fixed versions yet.
- Backup and restore are documented but not automated.
- No CI validation for Compose syntax yet.
- No `.env.example` file yet.
- Monitoring and alerting are not included.

## DevOps improvement roadmap

- Add `.env.example`
- Pin image versions instead of using `latest`
- Add backup and restore scripts
- Add Compose validation in GitHub Actions
- Add healthcheck documentation
- Add upgrade procedure with rollback notes
- Add basic monitoring examples

## Why this repository matters

For a DevOps portfolio, this repository demonstrates:

- Docker Compose service decomposition
- Reverse proxy integration through Traefik labels
- Private and public Docker network separation
- Persistent volume design
- Environment-based configuration
- Basic operational documentation
- Backup and restore planning
