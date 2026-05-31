# Django + Docker + Jenkins (CI/CD demo)

This repo is a **very simple Django app** packaged with **Docker**, intended primarily to demonstrate **Jenkins builds**.

- Django URL: `http://localhost:8000/django_app/`
- Admin URL: `http://localhost:8000/admin/`

## What’s in this repo

- Django project entrypoint: [manage.py](manage.py)
- Django project config: [django_prod/](django_prod/)
- Demo app + hello endpoint: [django_app/](django_app/)
- Container build/run: [Dockerfile](Dockerfile)
- Python deps: [requirements.txt](requirements.txt)

## Run locally (Docker)

Build:

```bash
docker build -t django-prod-app:local .
```

Run (foreground):

```bash
docker run --rm -p 8000:8000 django-prod-app:local
```

Run (background):

```bash
docker run -d --name django-prod-container -p 8000:8000 django-prod-app:local
```

Stop:

```bash
docker stop django-prod-container && docker rm django-prod-container
```

## Jenkins (Freestyle job) — Execute shell

This is the simplest Jenkins setup:

1. Create a **Freestyle project**
2. **Source Code Management**: configure Git repo (or keep “None” and copy workspace)
3. Add build step: **Execute shell**
4. Paste one of the scripts below

### Option A: CI only (build + check + migrate + test)

This is recommended for CI because it **does not start a long-running server**.

```sh
#!/bin/bash
set -euo pipefail

IMAGE="django-prod-app:${BUILD_NUMBER:-local}"

# Build image from Dockerfile in the workspace
docker build -t "$IMAGE" .

# Run Django commands inside one-off containers
docker run --rm "$IMAGE" python manage.py check
docker run --rm "$IMAGE" python manage.py migrate --noinput
docker run --rm "$IMAGE" python manage.py test
```

### Option B: CI + run container (simple deploy on the Jenkins agent)

Use this only if your Jenkins agent machine is where you want the app running.

```sh
#!/bin/bash
set -euo pipefail

IMAGE="django-prod-app:${BUILD_NUMBER:-local}"
CONTAINER="django-prod-container"

docker build -t "$IMAGE" .

docker run --rm "$IMAGE" python manage.py check
docker run --rm "$IMAGE" python manage.py migrate --noinput
docker run --rm "$IMAGE" python manage.py test

# Replace the running container
docker stop "$CONTAINER" 2>/dev/null || true
docker rm "$CONTAINER" 2>/dev/null || true

docker run -d --name "$CONTAINER" -p 8000:8000 "$IMAGE"
```

## Jenkins notes (important)

- The Jenkins agent must have **Docker installed** and the Jenkins user must be able to run `docker`.
  - If needed, change `docker ...` to `sudo docker ...`.
- Port `8000` must be free on the Jenkins agent if you use Option B.
- `python manage.py runserver` is a development server; for production you’d typically run Gunicorn/uWSGI behind a reverse proxy.

## Quick verification

After running Option B, open:

- `http://<jenkins-agent-host>:8000/django_app/`
