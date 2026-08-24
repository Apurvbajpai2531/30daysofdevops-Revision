# Day 04 — Nginx Reverse Proxy

This document covers adding an **Nginx reverse proxy** in front of TrackForge's backend and frontend services, giving the whole app a single entry point.

## Overview

Until Day 3, the app was reachable on two separate ports:
- Frontend (Flask) → `http://localhost:5000`
- Backend (FastAPI) → `http://localhost:8000`

That's fine for local dev, but it's not how a real deployment looks. Day 04 puts **Nginx** in front of both services so the app is reachable on a single port (`80`), with Nginx routing requests based on path:

- `/api/*` → backend
- everything else → frontend

The backend and frontend containers are no longer exposed to the host directly — only Nginx is. They talk to Nginx (and each other) over the internal Docker network.

---

## `nginx/nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend_upstream {
        server trackforge-backend:8000;
    }

    upstream frontend_upstream {
        server trackforge-frontend:5000;
    }

    server {
        listen 80;
        server_name localhost;

        client_max_body_size 20M;

        location /api/ {
            proxy_pass http://backend_upstream;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location / {
            proxy_pass http://frontend_upstream;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

**What it does:**
- Defines two upstreams — `backend_upstream` and `frontend_upstream` — pointing at the backend/frontend containers by **container name** (Docker's internal DNS resolves these on the shared network).
- Any request to `/api/...` is proxied to the backend.
- Every other request falls through to the frontend.
- Standard proxy headers (`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) are forwarded so the backend/frontend still see the real client IP and scheme, not Nginx's.

---

## Compose changes

`docker-compose.yml` gained a new `nginx` service, and the existing `backend`/`frontend` services switched from `ports` (host-exposed) to `expose` (internal-only):

```yaml
  backend:
    # ...
    expose:
      - "8000"

  frontend:
    # ...
    expose:
      - "5000"

  nginx:
    image: nginx:stable-alpine
    container_name: trackforge-nginx
    restart: unless-stopped
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
    depends_on:
      - backend
      - frontend
    networks:
      - trackforge-net
```

`expose` vs `ports`:
- `ports` publishes a container port to the **host machine** — reachable from outside Docker.
- `expose` only makes the port reachable to **other containers on the same network** — not from the host or the internet.

This means after this change, `localhost:5000` and `localhost:8000` no longer work directly — everything goes through `localhost` (port 80, via Nginx).

---

## Commands — Nginx (Day 04)

### 1. Create the nginx config directory
```bash
mkdir -p nginx
```

### 2. Rebuild and start the full stack (db, backend, frontend, nginx)
```bash
docker compose down
docker compose up -d --build
```

### 3. Check all services are up
```bash
docker compose ps
```

### 4. View Nginx logs
```bash
docker logs trackforge-nginx
docker logs -f trackforge-nginx   # follow live
```

### 5. Reload Nginx config without restarting the container (after editing nginx.conf)
```bash
docker exec trackforge-nginx nginx -s reload
```

### 6. Test the config for syntax errors before reloading
```bash
docker exec trackforge-nginx nginx -t
```

---

## Gotcha: port 80 already in use

On a machine with Apache pre-installed, `nginx` may fail to bind to port 80:

```
Error response from daemon: failed to set up container networking:
driver failed programming external connectivity on endpoint trackforge-nginx:
failed to bind host port 0.0.0.0:80/tcp: address already in use
```

Check what's holding the port:
```bash
sudo lsof -i :80
```

If it's Apache and you don't need it:
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```

Otherwise, map Nginx to a different host port instead, e.g. `"8080:80"` in `docker-compose.yml`, and access the app at `http://localhost:8080`.

---

## Suggested Project Structure

```
day-04/
├── nginx/
│   └── nginx.conf
└── docker-compose.yml
```

---

## What's next

Day 04 gets the app to a single entry point on plain HTTP. Future days will build on this:
- TLS/HTTPS once a real domain is available
- Rate limiting and basic auth at the Nginx layer
- CI/CD to automate builds and deploys (Day 05)
