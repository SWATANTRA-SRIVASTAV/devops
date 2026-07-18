# Real-Time Chat — Dockerized Deployment (DevOps Assignment)

A FastAPI WebSocket chat application, containerized behind an Nginx reverse
proxy, deployed to a cloud VM with an automated GitHub Actions CI/CD pipeline.

![Architecture](docs/architecture.svg)

## Project overview

Two containers on a private Docker bridge network:

| Service | Image | Role |
|---|---|---|
| `backend` | built from `Dockerfile` (python:3.11-slim) | FastAPI app serving the `/ws` WebSocket endpoint |
| `nginx` | `nginx:alpine` | Reverse proxy — serves the static frontend and proxies `/ws` to the backend |

Only Nginx is exposed to the host (port 80). The backend is reachable only
from inside the Docker network, which is the correct production posture —
no reason to expose an internal service port publicly.

## Issues found and how they were fixed

The repository as provided had three deliberate misconfigurations. All three
are the same underlying category of mistake — confusing "inside a container"
networking with "on my laptop" networking — which is worth calling out
explicitly if you're asked about it.

**1. Backend bound to `127.0.0.1` instead of `0.0.0.0` (`Dockerfile`)**
`CMD ["uvicorn", "main:app", "--host", "127.0.0.1", ...]` makes uvicorn only
accept connections that originate *inside its own container's network
namespace*. Nginx, in a separate container, is a different network namespace
entirely — its requests were being refused at the TCP level before FastAPI
ever saw them. Fix: bind to `0.0.0.0` so uvicorn listens on all interfaces,
including the internal Docker network interface Nginx connects through.

**2. Frontend volume mount commented out (`docker-compose.yml`)**
The line mounting `./frontend` into `/usr/share/nginx/html` was disabled, so
the Nginx container fell back to its own image's default landing page.
Fix: uncommented and enabled the bind mount (read-only, since Nginx only
needs to read these files) — `./frontend:/usr/share/nginx/html:ro`.

**3. WebSocket proxy target and missing upgrade headers (`nginx.conf`)**
Two separate bugs in the same block:
- `proxy_pass http://localhost:8000/ws;` — inside the Nginx container,
  `localhost` refers to the Nginx container itself, which has nothing
  listening on 8000. Docker Compose gives every service a DNS entry equal to
  its service name on the shared network, so the fix is
  `proxy_pass http://backend:8000/ws;`.
- The `Upgrade` and `Connection` headers were commented out. A WebSocket
  connection starts as a normal HTTP request with an `Upgrade: websocket`
  header; Nginx has to forward that header (and set `Connection: upgrade`)
  or it treats the request as ordinary HTTP and the client-side handshake
  fails, which is exactly the "stuck on Disconnected" symptom described in
  the assignment brief.

### Additional hardening beyond the minimum fix

- Added a `HEALTHCHECK` to the backend image and wired
  `depends_on: condition: service_healthy` in Compose, so Nginx only starts
  routing traffic once the backend is actually accepting connections —
  not just once the container process has started.
- `restart: unless-stopped` on both services, so they come back up after a
  crash or a host reboot (systemd starts the Docker daemon on boot, and
  Docker restarts containers with this policy).
- Explicit named bridge network (`chat-network`) instead of relying on the
  Compose default, for clarity when this grows past two services.
- Dropped the obsolete `version: '3.8'` key — modern Docker Compose (v2)
  ignores it and prints a warning; the Compose spec no longer uses it.

## How Docker networking works here

Docker Compose creates a private bridge network for the project and attaches
an embedded DNS server to it. Every service gets a DNS record equal to its
service name — that's why `nginx.conf` can say `proxy_pass
http://backend:8000` instead of hardcoding an IP: Docker resolves `backend`
to whatever internal IP that container currently has, which is what makes
containers restartable/replaceable without reconfiguring anything downstream.

`expose: ["8000"]` on the backend documents the port for other containers on
the network without publishing it to the host — there is deliberately no
`ports:` mapping for the backend, since nothing outside the Docker network
should talk to FastAPI directly.

## How the Nginx reverse proxy works

Two `location` blocks in `nginx.conf`:

- `location /` — serves files straight from `/usr/share/nginx/html` (the
  mounted `frontend/` directory), falling back to `index.html` for any path
  that doesn't match a real file (`try_files ... /index.html`), which is the
  standard pattern for a single-page app.
- `location /ws` — proxies to the backend and adds the headers required to
  keep a WebSocket connection alive: `proxy_http_version 1.1` (WebSocket
  requires HTTP/1.1), `Upgrade`/`Connection` for the protocol switch, and a
  long `proxy_read_timeout`/`proxy_send_timeout` (86400s) since a chat socket
  is meant to stay open indefinitely rather than timing out like a normal
  HTTP request.

## How CI/CD works

`.github/workflows/deploy.yml` triggers on every push to `main`:

1. Checks out the repo (mostly so the workflow file itself is versioned;
   the actual deploy pulls fresh code on the server).
2. SSHes into the server using credentials stored as GitHub Secrets
   (`SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`, `SERVER_PORT`).
3. On the server: `git fetch` + `git reset --hard origin/main` to sync the
   working copy, then `docker compose up -d --build` to rebuild any changed
   image and restart containers with zero manual steps.
4. Prunes dangling images so the VM's disk doesn't fill up over repeated
   deploys.
5. Runs a follow-up SSH step that checks container status and curls
   `localhost` on the server, so a broken deploy shows up as a failed
   GitHub Actions run instead of silently going live.

## Deploying it yourself

### 1. Local verification

```bash
git clone <your-fork-url>
cd devops
docker compose up -d --build
docker compose ps      # both services should show healthy/running
```
Visit `http://localhost` — the chat UI should load and show "Connected".
Open a second browser tab to confirm multi-user broadcast works.

### 2. Cloud VM setup (example: Oracle Cloud Free Tier)

1. Create an "Always Free" Ampere or E2.1.Micro instance (Ubuntu 22.04).
2. Open port 80 (and 22 for SSH) in both the VM's security list/NSG **and**
   the OS firewall (`sudo ufw allow 80,22/tcp`) — Oracle Cloud blocks
   traffic at the cloud level even if `iptables`/`ufw` on the box allows it.
3. SSH in and install Docker:
   ```bash
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker $USER   # log out/in after this
   sudo apt-get install -y docker-compose-plugin
   ```
4. Clone the repo onto the VM and bring it up once manually to confirm it
   works end to end, matching step 1 above but against the VM's public IP.

### 3. Wire up GitHub Actions

In your GitHub repo → **Settings → Secrets and variables → Actions**, add:

| Secret | Value |
|---|---|
| `SERVER_HOST` | VM's public IP |
| `SERVER_USER` | SSH username (e.g. `ubuntu`) |
| `SERVER_SSH_KEY` | Private key with access to the VM (generate a dedicated deploy key, don't reuse your personal key) |
| `SERVER_PORT` | `22` (or your custom SSH port) |

Push to `main` and watch the **Actions** tab — the workflow will SSH in,
pull, rebuild, and restart automatically from then on.

### 4. Confirm

```
http://<your-public-ip>
```

## Repository structure

```
devops/
├── .github/workflows/deploy.yml   # CI/CD pipeline
├── app/
│   ├── main.py                    # FastAPI app (unmodified)
│   └── requirements.txt
├── frontend/
│   └── index.html                 # chat UI (unmodified)
├── docs/
│   └── architecture.svg
├── Dockerfile                     # fixed: 0.0.0.0 bind + healthcheck
├── docker-compose.yml             # fixed: volume mount, network, healthy-dependency
├── nginx.conf                     # fixed: backend service name, upgrade headers
└── README.md
```

## Bonus items not implemented (out of scope for this pass)

HTTPS via Let's Encrypt, Redis-backed presence/state, Terraform IaC, and a
load-balancer writeup were left out to keep this submission focused on the
mandatory requirements — happy to add any of them on request.
