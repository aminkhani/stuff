## 🧱 Splitting Settings (base/dev/prod)

Don't use one `settings.py` with `if DEBUG:` branches everywhere — split into a package instead.

```
myproject/
└── settings/
    ├── __init__.py
    ├── base.py       # shared by everyone
    ├── dev.py        # local development
    └── production.py # production
```

```python
# base.py — shared settings live here (INSTALLED_APPS, MIDDLEWARE, TEMPLATES, etc.)
```

```python
# dev.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

DATABASES = {
    'default': {
        'ENGINE': 'django.contrib.gis.db.backends.postgis',
        'NAME': 'myproject_dev',
        'USER': 'dev_user',
        'PASSWORD': 'dev_password',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

INSTALLED_APPS += ['django_extensions']  # dev-only tools
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

```python
# production.py
import os
from .base import *

DEBUG = False
ALLOWED_HOSTS = os.environ['DJANGO_ALLOWED_HOSTS'].split(',')

DATABASES = {
    'default': {
        'ENGINE': 'django.contrib.gis.db.backends.postgis',
        'NAME': os.environ['DB_NAME'],
        'USER': os.environ['DB_USER'],
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': os.environ['DB_HOST'],
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}
```

Run with:

```bash
DJANGO_SETTINGS_MODULE=myproject.settings.dev python manage.py runserver
DJANGO_SETTINGS_MODULE=myproject.settings.production gunicorn myproject.wsgi
```

**✅ Why split?**

- No risk of a stray `DEBUG = True` sneaking into prod
- Dev-only tools (`django-debug-toolbar`, `django-extensions`) never ship to prod by accident
- Clear diff between environments when reviewing PRs

---
## 🔑 Secret Key & Environment Variables

Never hardcode secrets. Use `django-environ` or plain `os.environ` + a `.env` file (never committed).

```bash
pip install django-environ
```

```python
# base.py
import environ

env = environ.Env()
environ.Env.read_env()  # reads .env file

SECRET_KEY = env('SECRET_KEY')
DEBUG = env.bool('DEBUG', default=False)
```

```ini
# .env  (add to .gitignore!)
SECRET_KEY=django-insecure-change-me-in-real-env
DEBUG=True
DB_PASSWORD=supersecret
```

> [!warning] Rotate the secret key if it ever leaks `SECRET_KEY` signs sessions, password reset tokens, and CSRF tokens. A leaked key lets an attacker forge sessions. Generate a new one with:
> 
> ```python
> from django.core.management.utils import get_random_secret_key
> print(get_random_secret_key())
> ```

In production, prefer a **secrets manager** (AWS Secrets Manager, HashiCorp Vault, Doppler) over `.env` files on disk where possible.

---
## 🐞 DEBUG & ALLOWED_HOSTS

```python
# production.py
DEBUG = False
ALLOWED_HOSTS = ['api.example.com', 'www.example.com']
```

> [!warning] `DEBUG = True` in production is a critical vulnerability It exposes full stack traces, source code snippets, settings values, and installed packages to anyone who triggers a 500 error. This is one of the most common real-world Django security incidents.

`ALLOWED_HOSTS` prevents HTTP Host header attacks — Django rejects any request whose `Host` header doesn't match this list (ignored when `DEBUG=True`, which is another reason to keep `DEBUG=False` in prod).

---
## 🗄️ Database Config

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.contrib.gis.db.backends.postgis',  # PostGIS backend
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT', default='5432'),
        'CONN_MAX_AGE': 60,       # 🔁 persistent connections (connection pooling-lite)
        'OPTIONS': {
            'sslmode': 'require',  # 🔒 enforce SSL to Postgres in prod, see [[postgresql-config|PostgreSQL Config]]
        },
    }
}
```

> [!tip] Connection pooling `CONN_MAX_AGE` reuses connections across requests. For high-traffic prod, pair with `pgbouncer` in front of Postgres rather than relying solely on Django's built-in reuse.

---
## 🔒 Security Headers & HTTPS

```python
# production.py — force HTTPS and harden headers
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000          # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = 'DENY'

# If behind a reverse proxy (nginx, load balancer) terminating SSL:
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

|Setting|What it does|
|---|---|
|`SECURE_SSL_REDIRECT`|Redirects all HTTP → HTTPS|
|`SECURE_HSTS_SECONDS`|Tells browsers to _only_ use HTTPS for this domain going forward|
|`SECURE_CONTENT_TYPE_NOSNIFF`|Prevents browsers from MIME-sniffing responses|
|`X_FRAME_OPTIONS = 'DENY'`|Blocks the site from being embedded in an `<iframe>` (clickjacking protection)|

> [!warning] `SECURE_HSTS_PRELOAD` Only enable once you're 100% sure HTTPS works everywhere — HSTS preload is very hard to undo (browsers cache it for a long time, and preload lists take time to update).

---
## 🍪 Session & CSRF Cookies

```python
SESSION_COOKIE_SECURE = True      # only sent over HTTPS
CSRF_COOKIE_SECURE = True         # only sent over HTTPS
SESSION_COOKIE_HTTPONLY = True    # not accessible via JS (XSS protection)
CSRF_COOKIE_HTTPONLY = False      # Django needs this readable for the CSRF token pattern in most JS setups
SESSION_COOKIE_SAMESITE = 'Lax'   # or 'Strict' for tighter CSRF protection
CSRF_TRUSTED_ORIGINS = ['https://example.com', 'https://api.example.com']
```

---
## 📁 Static & Media Files

```python
# base.py
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'   # collected here by collectstatic

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

```python
# production.py — serve via WhiteNoise or cloud storage, never Django's dev server
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
# or for S3-backed media:
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_STORAGE_BUCKET_NAME = env('AWS_STORAGE_BUCKET_NAME')
```

> [!warning] Never use `runserver` or Django to serve static/media files in production Django's dev server is not built for performance or security at scale. Use WhiteNoise, nginx, or object storage (S3/GCS) + a CDN instead.

---
## 📝 Logging

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {'class': 'logging.StreamHandler', 'formatter': 'verbose'},
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': 'logs/django.log',
            'maxBytes': 1024 * 1024 * 10,  # 10MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
    },
    'root': {'handlers': ['console'], 'level': 'INFO'},
    'loggers': {
        'django': {'handlers': ['console', 'file'], 'level': 'INFO', 'propagate': False},
        'django.security': {'handlers': ['file'], 'level': 'WARNING', 'propagate': False},
    },
}
```

In production, ship logs to a centralized system (Sentry for errors, ELK/Datadog/CloudWatch for general logs) rather than relying only on local files.

---
## 🌐 CORS

```bash
pip install django-cors-headers
```

```python
INSTALLED_APPS += ['corsheaders']
MIDDLEWARE = ['corsheaders.middleware.CorsMiddleware', *MIDDLEWARE]

# dev.py
CORS_ALLOW_ALL_ORIGINS = True   # ✅ fine for local dev

# production.py
CORS_ALLOWED_ORIGINS = [
    'https://example.com',
    'https://app.example.com',
]
CORS_ALLOW_CREDENTIALS = True
```

> [!warning] Never use `CORS_ALLOW_ALL_ORIGINS = True` in production Combined with `CORS_ALLOW_CREDENTIALS = True`, this can expose authenticated endpoints to any website.

---
## ⚡ Celery / PostGIS notes

```python
# base.py — Celery config (shared)
CELERY_BROKER_URL = env('CELERY_BROKER_URL', default='redis://localhost:6379/0')
CELERY_RESULT_BACKEND = env('CELERY_RESULT_BACKEND', default='redis://localhost:6379/0')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
```

```python
# base.py — GIS app requirement
INSTALLED_APPS += ['django.contrib.gis']
```

- In production, Celery broker URLs should come from env vars/secrets manager too — same rule as DB credentials.
- See [[PostgreSQL Config|PostgreSQL Config & pg_hba]] for hardening the Postgres side (`sslmode=require` here pairs with `hostssl` there).

---
## ⚠️ Common Pitfalls

- 🚨 Shipping `DEBUG = True` to production — exposes stack traces and settings.
- 🚨 Committing `.env` or hardcoded `SECRET_KEY` to version control.
- 🚨 Forgetting `ALLOWED_HOSTS` in prod (Django will refuse all requests with a 400, or worse, be misconfigured to `['*']`).
- 🚨 Using SQLite in production for anything beyond a tiny/prototype app — no real concurrency support.
- 🚨 Serving static/media files via Django itself in production instead of WhiteNoise/CDN/object storage.
- 🚨 `CORS_ALLOW_ALL_ORIGINS = True` left on by copy-pasting dev settings into prod.
- 🚨 Not setting `CSRF_TRUSTED_ORIGINS` when frontend and backend are on different subdomains — CSRF-protected POSTs silently fail.

---
## ✅ Best Practices Checklist

- [ ] Settings split into `base.py` / `dev.py` / `production.py`
- [ ] `SECRET_KEY` and all credentials loaded from environment variables / secrets manager, never hardcoded
- [ ] `.env` in `.gitignore`
- [ ] `DEBUG = False` in production, verified via env var not a manual toggle
- [ ] `ALLOWED_HOSTS` explicitly set to real domains in production
- [ ] `SECURE_SSL_REDIRECT`, HSTS, `X_FRAME_OPTIONS`, `SECURE_CONTENT_TYPE_NOSNIFF` all enabled in production
- [ ] Session/CSRF cookies set to `Secure` + `HttpOnly` (where applicable) in production
- [ ] Static/media served via WhiteNoise, nginx, or object storage — never Django dev server
- [ ] `CORS_ALLOWED_ORIGINS` explicitly scoped in production (never `CORS_ALLOW_ALL_ORIGINS`)
- [ ] Postgres connection uses `sslmode=require` in production
- [ ] Centralized logging/error tracking configured (Sentry, CloudWatch, etc.)
- [ ] `CONN_MAX_AGE` / pgbouncer considered for connection pooling under load