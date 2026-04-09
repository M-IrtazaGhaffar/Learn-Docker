# 🐳 Learn Docker, Docker Compose (V2), and a Basic Touch of Kubernetes (K8s)

> This is a practical, “do-this-then-that” guide that covers **every day operations** for Docker + Compose, and then shows how those ideas map into **Kubernetes** (Pods, Deployments, Services, Ingress).

---

## 1) Docker — Core Concepts

Docker gives you a standard way to package your app + its runtime + dependencies into an **image**, and then run it as an isolated process called a **container**. Images are built from a **Dockerfile**. 

**Key objects:**
- **Image**: read-only template used to create containers. 
- **Container**: a running instance of an image (your app process).
- **Dockerfile**: build recipe; you typically define a default program via `CMD`. 

---

## 2) Docker — Daily Operations (Commands)

### 2.1 Install / Verify Docker
Use Docker’s official install method for your OS (Docker Desktop for Windows/macOS; Docker Engine for Linux). (“Get Docker” points to the right installer paths.) 

Verify:
```bash
docker --version
docker info
docker ps
```

---

### 2.2 Images — Pull, List, Inspect, Remove

**Pull an image:**
```bash
docker pull nginx:alpine
```

**List images:**
```bash
docker images
```

**Inspect image details:**
```bash
docker image inspect nginx:alpine
```

**Remove an image:**
```bash
docker rmi nginx:alpine
```

---

### 2.3 Containers — Run (Most Important Command)

The main command is `docker container run` (`docker run` is a shortcut). 

**Run a container (foreground):**
```bash
docker run nginx:alpine
```

**Run detached + name + ports:**
```bash
docker run -d --name web -p 8080:80 nginx:alpine
```

What those flags mean:
- `-d`: run in background
- `--name web`: easier to reference later
- `-p HOST:CONTAINER`: publish a container port to your machine (port mapping) 

**Run and auto-remove when it stops:**
```bash
docker run --rm -it ubuntu:24.04 bash
```

- `--rm` cleans up the stopped container automatically
- `-it` gives you an interactive shell

---

### 2.4 Container Lifecycle — Start/Stop/Restart/Remove

```bash
docker ps              # running containers
docker ps -a           # all containers (including stopped)

docker stop web
docker start web
docker restart web

docker rm web          # remove stopped container
docker rm -f web       # force remove (even if running)
```

---

### 2.5 Logs, Exec, Copy (Debug Like a Pro)

**View logs:**
```bash
docker logs web
docker logs -f web     # follow (live)
```

**Exec into a running container:**
```bash
docker exec -it web sh
# or bash if available:
docker exec -it web bash
```

`docker exec` needs a **container name/ID**, not an image name. 

**Copy files:**
```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./site.html web:/usr/share/nginx/html/site.html
```

---

### 2.6 Environment Variables

```bash
docker run -d --name api \
  -e NODE_ENV=production \
  -e PORT=5000 \
  -p 5000:5000 \
  my-api:latest
```

---

### 2.7 Resource Limits (Basic)

```bash
docker run -d --name worker \
  --memory="512m" \
  --cpus="1.0" \
  my-worker:latest
```

---

### 2.8 Cleanup (Disk Space)

```bash
docker system df
docker system prune -f          # remove unused containers/networks (careful)
docker image prune -af          # remove unused images (more aggressive)
docker volume prune -f          # remove unused volumes (very careful!)
```

---

## 3) Dockerfiles — Build Images Properly

Dockerfiles define repeatable builds; only the **last `CMD`** is used. 

### 3.1 A Solid Node.js Dockerfile (Multi-stage)

```dockerfile
# ---- deps ----
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# ---- build ----
FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# ---- runtime ----
FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist

EXPOSE 5000
CMD ["node", "dist/index.js"]
```

### 3.2 `.dockerignore` (Must Have)

Create `.dockerignore` to avoid copying junk into the image:
```gitignore
node_modules
dist
.git
.gitignore
Dockerfile
docker-compose.yml
npm-debug.log
.env
```

### 3.3 Build, Tag, Run

```bash
docker build -t my-api:1.0 .
docker run -d --name my-api -p 5000:5000 my-api:1.0
```

### 3.4 Dockerfile Best Practices (What People Miss)

Docker recommends using **curated Docker Official Images** and following best practices around image construction.   
Practical rules:
- Prefer small base images when possible (e.g., `alpine`) *if your deps support it*
- Use multi-stage builds for smaller runtime images
- Don’t bake secrets into images
- Keep layers cache-friendly (copy package files first, then install, then copy source)

---

## 4) Volumes & Persistence (Data)

Docker storage options:
- **Volumes** (managed by Docker, best for persistent data)
- **Bind mounts** (map a host path into container; great for dev)
- **tmpfs** (in-memory, ephemeral) 

### 4.1 Named Volume (Best for Databases)

```bash
docker volume create pgdata

docker run -d --name postgres \
  -e POSTGRES_PASSWORD=pass123 \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:latest
```

Volumes persist even if the container is removed. 

### 4.2 Bind Mount (Best for Local Dev)

```bash
docker run --rm -it \
  -v "$PWD":/app \
  -w /app \
  node:20-alpine \
  sh
```

Bind mounts directly link a host path to the container. 

---

## 5) Networking (Container-to-Container)

Docker networking lets containers talk to each other using container names on a user-defined network. Docker supports connecting containers to networks using `--network`. 

### 5.1 Create a Network and Attach Containers

```bash
docker network create app-net

docker run -d --name redis --network app-net redis:alpine
docker run -d --name api --network app-net -e REDIS_HOST=redis my-api:latest
```

---

## 6) Docker Compose (V2) — Multi-Container Apps

Compose is the standard way to define and run **multi-container** apps. Compose V2 is integrated into the Docker CLI as `docker compose`. 

### 6.1 Compose File Names
Compose automatically looks for:
- `compose.yaml` or
- `docker-compose.yaml` 

### 6.2 Common Commands

```bash
docker compose up -d          # start
docker compose ps             # status
docker compose logs -f        # logs
docker compose down           # stop + remove containers + network
docker compose down -v        # also remove volumes (careful)
docker compose build          # build images
docker compose pull           # pull images
docker compose exec api sh    # shell inside service container
```

### 6.3 Compose Networking (Default)
Compose creates a default network for your project; you can also configure custom or external networks. 

---

## 7) Compose — Full Example (Node API + Redis)

### 7.1 `compose.yaml`

```yaml
services:
  redis:
    image: redis:alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

  api:
    build: .
    container_name: api
    ports:
      - "5000:5000"
    environment:
      - PORT=5000
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    restart: unless-stopped

volumes:
  redis_data:
```

### 7.2 Run It

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f api
```

### 7.3 Stop It

```bash
docker compose down
# or wipe data too:
docker compose down -v
```

---

## 8) Basic Kubernetes Touch — The “Next Level”

Kubernetes is a container orchestrator. Instead of you manually running containers, Kubernetes runs workloads inside **Pods** (the smallest deployable unit) and manages them using controllers like **Deployments**. 

### 8.1 Core K8s Objects You Should Know

- **Pod**: one or more containers scheduled together.
- **Deployment**: manages stateless Pods, scaling, rolling updates; good fit for most web APIs. 
- **Service**: stable network endpoint in front of Pods (Pods are ephemeral). 
- **Ingress**: HTTP(S) routing into the cluster (requires an Ingress Controller). 
- **ConfigMap**: non-secret config injected into Pods.
- **Secret**: sensitive data, with important security cautions (etcd encryption / RBAC). 

### 8.2 Local Kubernetes Quickly (kind)

`kind` runs local Kubernetes clusters using Docker containers as nodes. 

Typical flow:
```bash
kind create cluster --name dev
kubectl cluster-info
kubectl get nodes
```

---

## 9) K8s — Minimal Example (Deployment + Service + Ingress)

> Example app: `my-api` container listens on `5000`.

### 9.1 Deployment (`deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
        - name: my-api
          image: my-api:1.0
          ports:
            - containerPort: 5000
          env:
            - name: NODE_ENV
              value: "production"
```

Deployments have required fields like `.apiVersion`, `.kind`, `.metadata`, and `.spec` with a pod template. 

Apply it:
```bash
kubectl apply -f deployment.yaml
kubectl get deploy
kubectl get pods
```

`kubectl apply` creates resources if they don’t exist and updates if they do. 

### 9.2 Service (`service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api-svc
spec:
  selector:
    app: my-api
  ports:
    - port: 80
      targetPort: 5000
  type: ClusterIP
```

A Service exposes a set of Pods behind a stable endpoint (important because Pods change). 

Apply:
```bash
kubectl apply -f service.yaml
kubectl get svc
```

### 9.3 Ingress (`ingress.yaml`) — HTTP Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-api-ingress
spec:
  rules:
    - host: my-api.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-api-svc
                port:
                  number: 80
```

Ingress is the HTTP entry point conceptually (requires an Ingress controller). 

Apply:
```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```

### 9.4 Day-to-Day K8s Ops

```bash
kubectl get all
kubectl describe pod <pod-name>
kubectl logs -f <pod-name>
kubectl exec -it <pod-name> -- sh

kubectl scale deploy my-api --replicas=5
kubectl rollout status deploy my-api
kubectl rollout history deploy my-api
kubectl rollout undo deploy my-api
```

---

## 10) Docker vs Compose vs Kubernetes (Mental Model)

| Tool | What it solves | Typical use |
|------|-----------------|-------------|
| Docker | Build/run **one container** | Local dev, simple services |
| Compose | Run **many containers together** with networks/volumes | Local dev, small servers |
| Kubernetes | Run **many containers across many machines** (scheduling, scaling, self-healing) | Production / microservices / HA |

Kubernetes runs workloads in Pods and uses Deployments/Controllers so you don’t manage Pods one-by-one. 

---

# Cheat Sheets

## Docker “Top 15” Commands
```bash
docker pull <img>
docker build -t <name:tag> .
docker images
docker run -d --name <c> -p HOST:CONT <img>
docker ps
docker ps -a
docker logs -f <c>
docker exec -it <c> sh
docker stop <c>
docker start <c>
docker restart <c>
docker rm -f <c>
docker volume ls
docker network ls
docker system prune -f
```

## Docker Compose (V2) “Top 10” Commands
```bash
docker compose up -d --build
docker compose ps
docker compose logs -f
docker compose exec <svc> sh
docker compose down
docker compose down -v
docker compose pull
docker compose build
docker compose restart
docker compose config
```

## Kubernetes “Starter” Commands
```bash
kubectl apply -f .
kubectl get pods,svc,deploy,ingress
kubectl describe pod <pod>
kubectl logs -f <pod>
kubectl exec -it <pod> -- sh
kubectl scale deploy <name> --replicas=3
kubectl rollout status deploy <name>
kubectl rollout undo deploy <name>
```

--- 

If you want, you can copy/paste this as your `README.md` and I can also generate:
- a **production-ready Compose** example (with NGINX reverse proxy + SSL + healthchecks), and
- a matching **Kubernetes version** (Deployment + Service + ConfigMap + Secret + Ingress) for the same app.
