🇬🇧 [English version](README.md)

# Déploiement sécurisé Docker & Kubernetes

Des fichiers de déploiement durcis pour une application web en trois couches : un reverse proxy nginx, une application web Python/Flask et une base PostgreSQL. Le dépôt contient deux versions de la même stack, une pour **Docker Compose** et une pour **Kubernetes (K3s)**.

C'est le TP final du cours de *Virtualisation et conteneurs* du Bachelor Cybersécurité de l'EPITA, en 2e année. L'objectif était de sécuriser un déploiement volontairement vulnérable.

> **À noter :** l'image de l'application web (`tp-app-web:alpine`) était fournie par le cours et **n'est pas** incluse dans ce dépôt. Il faut la construire ou l'importer soi-même avant de déployer.

## Ce qui a été durci

**Docker Compose (`secure/`)**
- Versions d'images figées (`nginx:1.28-alpine`, `postgres:15-bookworm`) au lieu de `:latest`
- nginx est le seul point d'entrée publié (`8080:80`). L'application n'est joignable qu'à travers nginx, et la base est sur un réseau `internal` sans accès Internet.
- Les secrets sont lus dans un fichier `.env` (voir `.env.example`), jamais écrits dans le compose
- Systèmes de fichiers en lecture seule (`read_only`) avec des `tmpfs` là où il faut, `cap_drop: ALL` avec seulement les capacités minimales rajoutées, et `no-new-privileges`
- Limites de CPU, de mémoire et de PID sur chaque service
- Des healthchecks, avec `depends_on` qui attend que les services soient sains
- nginx : `server_tokens off`, en-têtes de sécurité (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Content-Security-Policy`) et limitation du débit par IP (10 req/s, rafale de 20)

**Kubernetes (`k8s/`)**
- Un namespace dédié `tp` et un Secret `app-secrets`, injecté via `secretKeyRef`
- L'application tourne avec 3 réplicas, en `runAsNonRoot` (UID 1001), avec `allowPrivilegeEscalation: false`, un `readOnlyRootFilesystem` et toutes les capacités retirées
- Des demandes et limites de ressources, plus une sonde sur `/health`
- PostgreSQL n'est exposé qu'à l'intérieur du cluster (Service ClusterIP)
- Un Ingress Traefik devant l'application web

## Fichiers

```
secure/
├── docker-compose.yml     # stack durcie : nginx-proxy, web-app, db
├── nginx.conf             # reverse proxy + en-têtes + limitation du débit
└── .env.example           # variables à recopier dans .env
k8s/
├── 00-namespace-secret.yaml
├── 01-postgres.yaml       # Deployment + Service
├── 02-deployment.yaml     # Deployment de l'application web
├── 03-service.yaml
└── 04-ingress.yaml        # Traefik
```

Les commentaires dans les fichiers sont en français.

## Prérequis

- Docker avec Docker Compose v2, **ou** un cluster Kubernetes (écrit pour K3s, qui inclut Traefik)
- L'image `tp-app-web:alpine` disponible en local (voir la note plus haut)

## Utilisation

### Docker Compose

```bash
cd secure
cp .env.example .env      # puis remplace chaque "change-me" par tes propres valeurs
docker compose up -d
docker compose ps
```

L'application est ensuite accessible sur <http://localhost:8080>.

### Kubernetes (K3s)

Modifie `k8s/00-namespace-secret.yaml` pour remplacer les valeurs `change-me`, puis :

```bash
kubectl apply -f k8s/
kubectl -n tp get pods,svc,ingress
```

Avec K3s, importe d'abord l'image de l'application, par exemple avec `sudo k3s ctr images import tp-app-web.tar`.

## État d'avancement

Le déploiement est terminé, tel que rendu pour le cours. Limites connues, aussi signalées dans les fichiers :

### TODO
- [ ] Auditer et reconstruire l'image `tp-app-web:alpine`, qui venait du cours et n'était pas fiable telle quelle.
- [ ] Utiliser un PersistentVolumeClaim pour PostgreSQL dans Kubernetes. Les données sont pour l'instant dans un `emptyDir` et sont perdues au redémarrage du pod.
- [ ] Gérer le Secret Kubernetes en dehors du dépôt, par exemple avec Sealed Secrets ou un gestionnaire de secrets externe.
