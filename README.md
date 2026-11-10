🇫🇷 [Version française](README.fr.md)

# Secure Docker & Kubernetes Deployment

Hardened deployment files for a three-tier web application: an nginx reverse proxy, a Python/Flask web app and a PostgreSQL database. The repository has two versions of the same stack, one for **Docker Compose** and one for **Kubernetes (K3s)**.

This was the final practical of the *Container Virtualisation* course in the EPITA Cybersecurity Bachelor, year 2. The goal was to secure a deliberately weak deployment.

> **Note:** the web application image (`tp-app-web:alpine`) was provided by the course and is **not** included in this repository. You need to build or import it yourself before deploying.

## What was hardened

**Docker Compose (`secure/`)**
- Pinned image versions (`nginx:1.28-alpine`, `postgres:15-bookworm`) instead of `:latest`
- nginx is the only published entry point (`8080:80`). The app is only reachable through nginx, and the database is on an `internal` network with no Internet access.
- Secrets are read from a `.env` file (see `.env.example`), never written in the compose file
- `read_only` root filesystems with `tmpfs` where needed, `cap_drop: ALL` with only the minimal capabilities added back, and `no-new-privileges`
- CPU, memory and PID limits on every service
- Healthchecks, with `depends_on` waiting for healthy services
- nginx: `server_tokens off`, security headers (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Content-Security-Policy`) and per-IP rate limiting (10 req/s, burst 20)

**Kubernetes (`k8s/`)**
- A dedicated `tp` namespace and an `app-secrets` Secret, injected through `secretKeyRef`
- The web app runs with 3 replicas, `runAsNonRoot` (UID 1001), `allowPrivilegeEscalation: false`, a `readOnlyRootFilesystem` and all capabilities dropped
- Resource requests and limits, plus a `/health` probe
- PostgreSQL is exposed only inside the cluster (ClusterIP Service)
- A Traefik Ingress in front of the web app

## Files

```
secure/
├── docker-compose.yml     # hardened stack: nginx-proxy, web-app, db
├── nginx.conf             # reverse proxy + headers + rate limiting
└── .env.example           # variables to copy into .env
k8s/
├── 00-namespace-secret.yaml
├── 01-postgres.yaml       # Deployment + Service
├── 02-deployment.yaml     # web app Deployment
├── 03-service.yaml
└── 04-ingress.yaml        # Traefik
```

## Requirements

- Docker with Docker Compose v2, **or** a Kubernetes cluster (written for K3s, which ships Traefik)
- The `tp-app-web:alpine` image available locally (see the note above)

## Usage

### Docker Compose

```bash
cd secure
cp .env.example .env      # then replace every "change-me" with your own values
docker compose up -d
docker compose ps
```

The application is then available at <http://localhost:8080>.

### Kubernetes (K3s)

Edit `k8s/00-namespace-secret.yaml` and replace the `change-me` values, then:

```bash
kubectl apply -f k8s/
kubectl -n tp get pods,svc,ingress
```

With K3s, import the application image first, for example with `sudo k3s ctr images import tp-app-web.tar`.

## Status

The deployment is finished as handed in for the course. Known limits, which are also noted in the files:

### TODO
- [ ] Audit and rebuild the `tp-app-web:alpine` image, which came from the course and was not trusted as is.
- [ ] Use a PersistentVolumeClaim for PostgreSQL in Kubernetes. Data currently lives in an `emptyDir` and is lost when the pod restarts.
- [ ] Manage the Kubernetes Secret outside the repository, for example with Sealed Secrets or an external secrets manager.
