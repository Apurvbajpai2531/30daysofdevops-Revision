# Day 02 — Dockerizing Frontend (TrackForge Flask UI)

This document covers the multi-stage `Dockerfile` used to containerize the **frontend** (Flask, server-rendered) service.

## Overview

The frontend uses the same **multi-stage build** pattern as the backend:

1. **Stage 1 (`builder`)** — installs Python dependencies into an isolated user directory (`--user`), using the full `python:3.12` image (has build tools available).
2. **Stage 2 (`runtime`)** — copies only the installed packages from the builder stage into a lightweight `python:3.12-slim` image, keeping the final image small.

> Note: this "frontend" is a server-rendered Flask (Jinja2) app, not a JS-based UI (React/Vue/etc). If you ever swap it for a Node-based frontend, it would need its own `Dockerfile` (e.g. `node:20-alpine` build → static file server or `npm run start`).

---

## `frontend/Dockerfile`

```dockerfile
# ---------- Stage 1: builder ----------
FROM python:3.12 AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ---------- Stage 2: runtime ----------
FROM python:3.12-slim

WORKDIR /app

COPY --from=builder /root/.local /root/.local

ENV PATH=/root/.local/bin:$PATH

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

**What it does:**
- Installs dependencies from `requirements.txt` in the builder stage.
- Copies compiled/installed packages (`/root/.local`) into the slim runtime image.
- Adds `/root/.local/bin` to `PATH` so installed console scripts are usable.
- Copies application source code.
- Exposes port `5000`.
- Runs the Flask app directly with `python app.py`.

---

## Commands — Frontend (Day 02)

### 1. Build the image
```bash
cd frontend
docker build -t trackforge-frontend:latest .
```

### 2. Rebuild without cache (force a fresh build)
```bash
docker build --no-cache -t trackforge-frontend:latest .
```

### 3. Run the container (standalone, quick test)
```bash
docker run -p 5000:5000 trackforge-frontend:latest
```

### 4. Run the container (production-style — detached, named, on custom network, auto-restart, env file)
```bash
docker run -d \
  --name trackforge-frontend \
  --network trackforge-net \
  --restart unless-stopped \
  -p 5000:5000 \
  -e API_BASE_URL="http://trackforge-backend:8000/api" \
  trackforge-frontend:latest
```

### 5. Stop and remove the container (before rebuilding/recreating)
```bash
docker stop trackforge-frontend
docker rm trackforge-frontend
```

### 6. View logs
```bash
docker logs trackforge-frontend
docker logs trackforge-frontend --tail 50
docker logs -f trackforge-frontend   # follow live
```

### 7. Check container status
```bash
docker ps -a --filter "name=trackforge-frontend"
```

### 8. Inspect environment variables inside the running container
```bash
docker exec trackforge-frontend env
```

### 9. Open a shell inside the running container (debugging)
```bash
docker exec -it trackforge-frontend /bin/bash
```

### 10. Update restart policy on an existing container
```bash
docker update --restart unless-stopped trackforge-frontend
```

### 11. Full rebuild-and-redeploy cycle (after code changes)
```bash
cd frontend
docker build -t trackforge-frontend:latest .
docker stop trackforge-frontend
docker rm trackforge-frontend
docker run -d \
  --name trackforge-frontend \
  --network trackforge-net \
  --restart unless-stopped \
  -p 5000:5000 \
  -e API_BASE_URL="http://trackforge-backend:8000/api" \
  trackforge-frontend:latest
```

---

## Networking Note

The frontend and backend must be on the same custom Docker network (`trackforge-net`) so the frontend can reach the backend by **container name** instead of `localhost`:

```bash
docker network create trackforge-net   # only if it doesn't already exist
docker network inspect trackforge-net  # verify both containers are attached
```

`API_BASE_URL` should point to `http://trackforge-backend:8000/api` — never `http://localhost:8000/api`, since inside a container `localhost` refers to that container itself, not the backend.

---

## Suggested Project Structure

```
day-02/
└── frontend/
    ├── Dockerfile
    ├── requirements.txt
    ├── app.py
    ├── templates/
    └── static/
````



