# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## What this repository is
This repo is a Docker-based local LNMP/LNMP+ stack template (Nginx/OpenResty + multiple PHP versions + MySQL/Redis/etc). It is mainly infrastructure/configuration, not an application codebase with its own test suite.

Primary files:
- `docker-compose.sample.yml`: service graph and container wiring
- `env.sample`: environment variables for ports, mounted paths, versions, extensions
- `services/*`: per-service Dockerfiles and runtime configs
- `www/`: mounted web root (contains sample `www/localhost/index.php`)

## Bootstrap and daily commands
1. Create local runtime files:
```bash
cp env.sample .env
cp docker-compose.sample.yml docker-compose.yml
```

2. Start core services:
```bash
docker compose up -d nginx php82 mysql
```

3. Rebuild a service after config/image changes:
```bash
docker compose build php82
docker compose up -d php82
```

4. Stop/remove stack:
```bash
docker compose down
```

## Common operational commands
- Service status:
```bash
docker compose ps
```

- Follow logs for key services:
```bash
docker compose logs -f nginx php82 mysql
```

- Enter containers:
```bash
docker exec -it nginx /bin/sh
docker exec -it php82 /bin/bash
docker exec -it mysql /bin/bash
```

- Reload Nginx after editing `services/nginx/*`:
```bash
docker exec -it nginx nginx -s reload
```

## Validation/lint/test commands for this repo
There is no built-in unit/integration test framework in this repository itself (no `phpunit.xml`, `composer.json`, CI test workflow, or lint config in-tree). Use these infra checks instead:

- Compose config validation:
```bash
docker compose config
```

- Nginx config validation (single “test”-style check):
```bash
docker exec -it nginx nginx -t
```

- PHP runtime sanity check:
```bash
docker exec -it php82 php -v
docker exec -it php82 php -m
```

- Endpoint smoke test:
```bash
curl -I http://localhost
curl -kI https://localhost
```

If you mount a real PHP app under `www/`, run that app’s own test/lint commands inside the appropriate PHP container.

## High-level architecture (big picture)
### Request/data flow
1. Host requests `localhost:80/443` reach `nginx` container.
2. `services/nginx/conf.d/localhost.conf` serves files from `/www/localhost`.
3. `location ~ \.php` forwards to `php82:9000` via FastCGI.
4. PHP code reaches databases/caches via Docker network service names (e.g., `mysql`, `redis`) rather than `localhost`.

### Configuration layering
- `docker-compose.yml` (copied from sample) defines enabled services and mounted files.
- `.env` controls versions, ports, extension lists, and host paths.
- Service-specific config lives under `services/<service>/...` and is bind-mounted into containers.
- Persistent data and logs are host-mounted to `data/*` and `logs/*`.

### Service model
- Default web path is bind-mounted from `${SOURCE_DIR}` (`./www`) to `/www`.
- PHP service images are custom-built from `services/php*/Dockerfile`.
- Optional services are present but commented in compose (Redis, MongoDB, OpenResty, RabbitMQ, Elastic stack, etc.); enable by uncommenting their blocks.

### Important coupling to remember
- Nginx PHP upstream is hardcoded in `services/nginx/conf.d/localhost.conf` (`fastcgi_pass php82:9000`).
- If switching PHP version, update `fastcgi_pass` to matching service name (e.g., `php74:9000`) and reload Nginx.
- PHP extension set is controlled by `.env` (`PHP82_EXTENSIONS`, etc.); image rebuild is required after extension changes.
