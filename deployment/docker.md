# Docker Deployment

## Overview

VoltWise runs as five Docker containers orchestrated by Docker Compose. Nginx sits at the edge and routes traffic to the two backend services. PostgreSQL and Redis provide persistence and caching. All containers share a private Docker network (`voltwise`) and are isolated from the host except through the proxy.

```
                        ┌─────────────┐
  Browser / Charger ───▶│    Nginx    │  :80 / :443
                        └──────┬──────┘
                               │
             ┌─────────────────┴──────────────────┐
             │                                    │
        /api/ /admin/                          /ocpp/
             │                                    │
    ┌────────▼────────┐                 ┌─────────▼────────┐
    │  voltwise-cloud  │                 │  voltwise-ocpp   │
    │  Django + WSGI   │                 │  Go WebSocket    │
    │  Gunicorn :8000  │                 │  Server :8080    │
    └────────┬─────────┘                 └──────────────────┘
             │
     ┌───────┴────────┐
     │                │
┌────▼─────┐   ┌──────▼──────┐
│PostgreSQL│   │    Redis    │
│   :5432  │   │    :6379    │
└──────────┘   └─────────────┘
```

## Services

| Service | Image / Source | Internal Port | Purpose |
|---------|---------------|---------------|---------|
| `db` | `postgres:17-alpine` | 5432 | Relational database — sessions, chargers, billing |
| `redis` | `redis:7-alpine` | 6379 | Cache and Celery broker |
| `cloud` | `docker/cloud/Dockerfile` | 8000 | Django REST API + admin |
| `ocpp` | `docker/ocpp/Dockerfile` | 8080 | OCPP 1.6 WebSocket server |
| `nginx` | `nginx:1.27-alpine` | **80, 443** | Reverse proxy — only public-facing service |

## File Structure

```
voltwise/
├── docker-compose.yml
├── .env.example
└── docker/
    ├── cloud/
    │   ├── Dockerfile      # Python 3.12 multi-stage build
    │   └── entrypoint.sh   # migrate → collectstatic → gunicorn
    ├── ocpp/
    │   └── Dockerfile      # Go 1.26 multi-stage build
    └── nginx/
        └── nginx.conf      # routing + WebSocket upgrade
```

## How to Run

### 1. Clone and configure

```bash
git clone <repo-url> voltwise
cd voltwise

# Create your local environment file
cp .env.example .env
```

Edit `.env` and set at minimum:

- `POSTGRES_PASSWORD` — change from default
- `SECRET_KEY` — generate with:
  ```bash
  python -c "import secrets; print(secrets.token_urlsafe(50))"
  ```
- `INTERNAL_API_KEY` — shared secret between services

### 2. Build and start

```bash
docker-compose up --build
```

First run will:
1. Pull base images (Postgres, Redis, Nginx)
2. Build `cloud` and `ocpp` images
3. Start `db` and `redis` (with health checks)
4. Run Django migrations automatically
5. Collect static files
6. Start Gunicorn and the Go server
7. Start Nginx last

### 3. Verify

```bash
# All containers running
docker-compose ps

# Django API health
curl http://localhost/api/

# OCPP WebSocket health check
curl http://localhost/healthz

# Django admin
open http://localhost/admin/
```

### 4. Create a Django superuser

```bash
docker-compose exec cloud python manage.py createsuperuser
```

### 5. Tear down

```bash
# Stop containers (data preserved)
docker-compose down

# Stop and remove volumes (wipes database)
docker-compose down -v
```

## URLs

| Endpoint | Description |
|----------|-------------|
| `http://localhost/api/` | Django REST API |
| `http://localhost/admin/` | Django admin panel |
| `ws://localhost/ocpp/<charger-id>` | OCPP 1.6 WebSocket endpoint |
| `http://localhost/healthz` | OCPP service health check |

## Configuration Reference

All environment variables are documented in `.env.example`.

Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `SECRET_KEY` | *(required)* | Django secret key |
| `DJANGO_SETTINGS_MODULE` | `config.settings.dev` | Switch to `config.settings.prod` for production |
| `POSTGRES_PASSWORD` | `voltwise_change_me_in_production` | Database password |
| `REDIS_URL` | `redis://redis:6379/0` | Redis connection string |
| `OCPP_LOG` | `info` | Log level for the Go server (`debug`, `info`, `warn`, `error`) |
| `GUNICORN_WORKERS` | `2` | Number of Gunicorn worker processes |

## Future Improvements

### SSL / TLS (Let's Encrypt)

The Nginx configuration already includes a commented-out `443` server block. To enable HTTPS:

**Option A — Certbot sidecar:**
```yaml
certbot:
  image: certbot/certbot
  volumes:
    - ./docker/nginx/certs:/etc/letsencrypt
  command: certonly --webroot -w /var/www/html -d yourdomain.com
```

**Option B — Replace Nginx with Caddy**, which handles certificates automatically:
```yaml
nginx:
  image: caddy:2-alpine
  volumes:
    - ./docker/nginx/Caddyfile:/etc/caddy/Caddyfile
```

### Production Settings

Switch `DJANGO_SETTINGS_MODULE=config.settings.prod` and configure:

1. A real domain in `ALLOWED_HOSTS`
2. Valid `CORS_ALLOWED_ORIGINS`
3. SSL certificates (Nginx HTTPS block)
4. `SECURE_SSL_REDIRECT = False` in `prod.py` — Nginx handles the redirect

### Scaling

Scale Django workers without changing `docker-compose.yml`:

```bash
docker-compose up --scale cloud=3
```

Add an upstream load-balancer block to `nginx.conf`:

```nginx
upstream cloud {
    least_conn;
    server cloud:8000;
    # Additional replicas are discovered via Docker DNS round-robin
}
```

### CI/CD

Suggested GitHub Actions pipeline:

```
push → build images → push to Container Registry → SSH deploy → docker-compose pull && up -d
```

### Celery Workers

Redis is already running. Add a worker service when ready:

```yaml
worker:
  build:
    context: .
    dockerfile: docker/cloud/Dockerfile
  command: celery -A config.celery worker -l info
  env_file: .env
  depends_on:
    - redis
    - db
  networks:
    - voltwise
```

### Observability

- Add **Prometheus** + **Grafana** for metrics
- Add **Sentry** DSN to Django settings for error tracking
- Enable structured JSON logging in Django (already active in the Go service)
