# Day 01 — Dockerizing Backend (TrackForge API)

This document covers the multi-stage `Dockerfile` used to containerize the **backend** (FastAPI) service.

## Overview

The backend uses a **multi-stage build**:

1. **Stage 1 (`builder`)** — installs Python dependencies into an isolated user directory (`--user`), using the full `python:3.12` image (has build tools available).
2. **Stage 2 (`runtime`)** — copies only the installed packages from the builder stage into a lightweight `python:3.12-slim` image, keeping the final image small.

This pattern avoids shipping compilers/build dependencies in the final image, reducing size and attack surface.

---

## `backend/Dockerfile`

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

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**What it does:**
- Installs dependencies from `requirements.txt` in the builder stage.
- Copies compiled/installed packages (`/root/.local`) into the slim runtime image.
- Adds `/root/.local/bin` to `PATH` so installed console scripts are usable.
- Copies application source code.
- Exposes port `8000`.
- Runs the FastAPI app with **Uvicorn**, serving the `app` object inside `app/main.py`.

---

## Commands — Backend (Day 01)

### 1. Build the image
```bash
cd backend
docker build -t trackforge-backend:latest .
```

### 2. Rebuild without cache (force a fresh build)
```bash
docker build --no-cache -t trackforge-backend:latest .
```

### 3. Create the custom network (only once, if not already created)
```bash
docker network create trackforge-net
```

### 4. Run the container (standalone, quick test)
```bash
docker run -p 8000:8000 trackforge-backend:latest
```

### 5. Run the container (production-style — detached, named, on custom network, auto-restart, env file)
```bash
docker run -d \
  --name trackforge-backend \
  --network trackforge-net \
  --restart unless-stopped \
  -p 8000:8000 \
  --env-file .env \
  trackforge-backend:latest
```

### 6. Stop and remove the container (before rebuilding/recreating)
```bash
docker stop trackforge-backend
docker rm trackforge-backend
```

### 7. View logs
```bash
docker logs trackforge-backend
docker logs trackforge-backend --tail 50
docker logs -f trackforge-backend   # follow live
```

### 8. Check container status
```bash
docker ps -a --filter "name=trackforge-backend"
```

### 9. Inspect environment variables inside the running container
```bash
docker exec trackforge-backend env
```

### 10. Open a shell inside the running container (debugging)
```bash
docker exec -it trackforge-backend /bin/bash
```

### 11. Update restart policy on an existing container
```bash
docker update --restart unless-stopped trackforge-backend
```

### 12. Full rebuild-and-redeploy cycle (after code changes)
```bash
cd backend
docker build -t trackforge-backend:latest .
docker stop trackforge-backend
docker rm trackforge-backend
docker run -d \
  --name trackforge-backend \
  --network trackforge-net \
  --restart unless-stopped \
  -p 8000:8000 \
  --env-file .env \
  trackforge-backend:latest
```

---

## Suggested Project Structure

```
day-01/
└── backend/
    ├── Dockerfile
    ├── requirements.txt
    ├── .env
    └── app/
        └── main.py
```