---
name: django-production-deployment
description: >
  Apply this skill for all production deployment work: Docker, docker-compose,
  Gunicorn/Uvicorn configuration, Nginx, static files, health checks, CI/CD pipeline,
  environment management, zero-downtime deployments, logging, observability,
  and production readiness checklist. Use when setting up infrastructure,
  writing Dockerfiles, configuring CI/CD, or doing a production readiness audit.
---

# Django Production Deployment Skill

> Production means real users, real data, real consequences.
> "It works on my machine" is not sufficient.
>
> FIRST: Use Context7 with library ID `/websites/djangoproject_en_6_0`
> for current Django deployment docs. For Gunicorn/Uvicorn configs,
> search Context7 for `gunicorn` and `uvicorn` before writing server config.

---

## 1. Production Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Internet / CDN                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                    Load Balancer                            │
│              (AWS ALB / Nginx / Caddy)                      │
└──────────┬─────────────────────────┬────────────────────────┘
           │                         │
┌──────────▼───────────┐  ┌──────────▼───────────┐
│  Django (HTTP)       │  │  Django (WebSocket)   │
│  Gunicorn + sync     │  │  Uvicorn/Daphne ASGI  │
│  workers             │  │  workers              │
└──────────┬───────────┘  └──────────┬────────────┘
           │                         │
┌──────────▼─────────────────────────▼────────────┐
│           Shared Services Layer                  │
│   PostgreSQL │ Redis │ Celery Workers            │
│   Celery Beat │ S3/Minio (media/static)          │
└──────────────────────────────────────────────────┘
```

---

## 2. Dockerfile — Production Grade

```dockerfile
# Multi-stage build — small final image
# Stage 1: Build dependencies
FROM python:3.13-slim AS builder

WORKDIR /build

# Install system deps for Python packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev libmagic1 \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps into a virtual environment
COPY requirements/production.txt .
RUN python -m venv /venv && \
    /venv/bin/pip install --upgrade pip && \
    /venv/bin/pip install --no-cache-dir -r production.txt

# Stage 2: Final image
FROM python:3.13-slim AS final

# Create non-root user
RUN groupadd --gid 1000 app && \
    useradd --uid 1000 --gid app --shell /bin/bash --create-home app

WORKDIR /app

# Copy only what's needed
COPY --from=builder /venv /venv
COPY --chown=app:app . .

# Environment
ENV PATH="/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1

# Static files
RUN /venv/bin/python manage.py collectstatic --noinput --settings=config.settings.production

USER app

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')"

CMD ["gunicorn", "config.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--worker-class", "sync", \
     "--worker-connections", "1000", \
     "--max-requests", "1000", \
     "--max-requests-jitter", "50", \
     "--timeout", "30", \
     "--keep-alive", "5", \
     "--log-level", "info", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

---

## 3. docker-compose.yml — Development

```yaml
# docker-compose.yml
version: "3.9"

x-django-base: &django-base
  build:
    context: .
    target: final
  env_file: .env
  depends_on:
    db:
      condition: service_healthy
    redis:
      condition: service_healthy
  volumes:
    - .:/app

services:
  web:
    <<: *django-base
    command: python manage.py runserver 0.0.0.0:8000
    ports:
      - "8000:8000"

  worker:
    <<: *django-base
    command: celery -A config.celery worker -l info -Q default,emails,notifications -c 4

  beat:
    <<: *django-base
    command: celery -A config.celery beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler

  flower:
    <<: *django-base
    command: celery -A config.celery flower --port=5555
    ports:
      - "5555:5555"

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-app_db}
      POSTGRES_USER: ${POSTGRES_USER:-app_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-app_pass}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER:-app_user}"]
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

---

## 4. Health Check Endpoint

```python
# apps/core/views.py
from django.http import JsonResponse, HttpRequest
from django.db import connection, OperationalError
from django.core.cache import cache
from django.views.decorators.http import require_GET
from django.views.decorators.cache import never_cache
import time


@require_GET
@never_cache
def health_check(request: HttpRequest) -> JsonResponse:
    """
    Health check endpoint for load balancer and container orchestration.
    Returns 200 if all critical services are available, 503 otherwise.
    """
    checks: dict[str, str] = {}
    healthy = True

    # Database
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        checks["database"] = "ok"
    except OperationalError:
        checks["database"] = "error"
        healthy = False

    # Cache / Redis
    try:
        key = "_health_check"
        cache.set(key, "1", timeout=5)
        assert cache.get(key) == "1"
        checks["cache"] = "ok"
    except Exception:
        checks["cache"] = "error"
        healthy = False

    status_code = 200 if healthy else 503
    return JsonResponse(
        {"status": "healthy" if healthy else "unhealthy", "checks": checks},
        status=status_code,
    )


@require_GET
@never_cache
def readiness_check(request: HttpRequest) -> JsonResponse:
    """
    Readiness probe — is the app ready to serve traffic?
    Kubernetes uses this to gate traffic.
    """
    return JsonResponse({"status": "ready"})
```

---

## 5. Gunicorn / Uvicorn Configuration

```python
# gunicorn.conf.py
import multiprocessing

# Workers
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "sync"  # Use "uvicorn.workers.UvicornWorker" for ASGI
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 50

# Timeouts
timeout = 30
keepalive = 5
graceful_timeout = 30

# Logging
loglevel = "info"
accesslog = "-"
errorlog = "-"
access_log_format = '{"remote_ip":"%(h)s","method":"%(m)s","path":"%(U)s","status":%(s)s,"duration":%(D)s}'

# Security
limit_request_line = 4094
limit_request_fields = 100
limit_request_field_size = 8190

# For ASGI (WebSocket support), use Uvicorn workers:
# gunicorn config.asgi:application \
#   --worker-class uvicorn.workers.UvicornWorker \
#   --workers 4
```

---

## 6. Nginx Configuration

```nginx
# nginx/nginx.conf
upstream django_http {
    server web:8000;
    keepalive 32;
}

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate     /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Body size limit
    client_max_body_size 10M;

    # Static files — serve directly, bypass Django
    location /static/ {
        alias /app/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        gzip_static on;
    }

    location /media/ {
        alias /app/media/;
        expires 7d;
        add_header Cache-Control "public";
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://django_http;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    # Django app
    location / {
        proxy_pass http://django_http;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # Enable keepalive
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

---

## 7. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
      redis:
        image: redis:7-alpine
        options: --health-cmd "redis-cli ping"

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
          cache: "pip"

      - name: Install dependencies
        run: |
          pip install -r requirements/testing.txt

      - name: Lint (ruff)
        run: ruff check .

      - name: Format check (ruff)
        run: ruff format --check .

      - name: Type check (mypy)
        run: mypy apps/

      - name: Run tests
        env:
          DJANGO_SETTINGS_MODULE: config.settings.testing
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
          DJANGO_SECRET_KEY: ci-test-secret-key
          JWT_SIGNING_KEY: ci-jwt-signing-key
        run: |
          pytest --cov --cov-report=xml

      - name: Security audit
        run: pip-audit

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
```

---

## 8. Static Files — Production

```python
# config/settings/production.py
# Use whitenoise for serving static files (no Nginx needed for simple setups)
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",  # Right after SecurityMiddleware
    ...
]

STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",  # Media files → S3
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}

# S3 settings (use django-storages)
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")
AWS_S3_REGION_NAME = env("AWS_S3_REGION_NAME", default="us-east-1")
AWS_S3_CUSTOM_DOMAIN = env("AWS_S3_CUSTOM_DOMAIN", default="")  # CloudFront domain
AWS_DEFAULT_ACL = "private"
AWS_S3_FILE_OVERWRITE = False
AWS_QUERYSTRING_AUTH = True  # Signed URLs for private files
AWS_QUERYSTRING_EXPIRE = 3600  # 1 hour signed URL expiry
```

---

## 9. Observability — Logs, Metrics, Traces

```python
# Structured JSON logging for log aggregation (DataDog, CloudWatch, ELK)
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "format": "%(asctime)s %(name)s %(levelname)s %(message)s %(request_id)s",
        },
    },
    "filters": {
        "require_debug_false": {"()": "django.utils.log.RequireDebugFalse"},
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "json"},
        "mail_admins": {
            "class": "django.utils.log.AdminEmailHandler",
            "filters": ["require_debug_false"],
            "level": "ERROR",
        },
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django": {"handlers": ["console"], "propagate": False},
        "apps": {"handlers": ["console"], "level": "DEBUG"},
    },
}
```

---

## 10. Production Readiness Checklist

```
Security:
✅ DEBUG = False
✅ ALLOWED_HOSTS explicitly set
✅ All secrets in environment variables
✅ HTTPS enforced + HSTS configured
✅ CSP headers configured (Django 6 native)
✅ Admin URL is NOT /admin/
✅ pip-audit passing in CI

Reliability:
✅ Health check endpoint (/health/)
✅ Graceful shutdown (SIGTERM handler)
✅ Database connection pooling (CONN_MAX_AGE)
✅ Celery worker retry policies configured
✅ Dead letter queue for failed tasks
✅ transaction.on_commit() for post-commit tasks
✅ Idempotent task design

Performance:
✅ Database query optimization (no N+1 in profiling)
✅ Redis caching for expensive queries
✅ Static files via CDN or whitenoise
✅ Media files on S3 (not local filesystem)
✅ Gzip/Brotli compression

Observability:
✅ Structured JSON logging
✅ Sentry error tracking
✅ Celery Flower monitoring
✅ Database slow query logging (pg_stat_statements)
✅ Request ID in all log lines

Deployment:
✅ Dockerized with non-root user
✅ Multi-stage Dockerfile (small final image)
✅ Database migrations run before traffic starts
✅ Zero-downtime deployment strategy
✅ Rollback procedure documented and tested
✅ CI/CD pipeline with automated tests + linting
✅ Environment parity (dev ≈ staging ≈ production)

Data:
✅ Automated database backups
✅ Backup restore tested (at least monthly)
✅ Soft deletes for critical records
✅ Data retention policy implemented
```
