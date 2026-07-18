# Docker

This guide walks through Docker step by step. Each section explains **why** something matters and **how** to do it with commands you can run on your machine.

---

## 1. What is Docker? What is a container?

**Container (concept):** A standardized way to **package an application together with its dependencies and configuration**, so it runs the same way on a laptop, a CI server, or production.

**Docker (product):** A platform built around the container idea: **build** images, **run** containers, **push/pull** from registries, and **orchestrate** (often with Kubernetes or Swarm—outside this baseline).


| Idea                       | Plain language                                                       |
| -------------------------- | -------------------------------------------------------------------- |
| **Portable artifact**      | You ship a known-good bundle instead of a long install document.     |
| **Efficient dev & deploy** | Same image in dev/stage/prod reduces “works on my machine” problems. |


Containers separate **“what the app needs”** (inside the image) from **“what the machine already has”** (mostly: a container runtime).

---

## 2. Where do images and containers “live”?

**Images** (read-only templates) are stored in **container registries**:


| Registry                           | Typical use                           |
| ---------------------------------- | ------------------------------------- |
| **Docker Hub** (`docker.io`)       | Public defaults; many official images |
| **Amazon ECR**                     | Private images on AWS                 |
| **Azure ACR**                      | Private images on Azure               |
| **Google GCR / Artifact Registry** | Private images on GCP                 |
| **GitHub GHCR**                    | Images next to your Git repo          |


Registries can be **public** (anyone can pull) or **private** (auth required).

**Container** = a **running instance** created **from** an image. The image lives in a registry; the running container lives on **your machine’s Docker engine** (until you delete it).

---

## 3. Life before containers (pain points)

**How people used to work:** Install databases, runtimes, and tools **directly on each machine** (MongoDB, Redis, MySQL, Node, Python, …).

**Why that broke down:**

1. **Different OSes** → different install steps and paths.
2. **Long checklists** → many steps where something goes wrong.
3. **Servers needed manual configuration** → “snowflake” servers.
4. **Deployment was often a text guide** → easy to misread or skip a step.
5. **Dependency version conflicts** → two apps need different library versions on the same host.
6. **Misunderstandings between dev and ops** → “I thought Redis was 6.x on prod.”

**After containers (what changes):** Teams collaborate on a **buildable artifact** (image). The server mainly needs a **container runtime** (e.g. Docker Engine / containerd). Runtime config is smaller and more repeatable.

---

## 4. Image vs container (critical distinction)


|               | **Image**                                          | **Container**                                                      |
| ------------- | -------------------------------------------------- | ------------------------------------------------------------------ |
| **What**      | Snapshot / recipe + layers (filesystem + metadata) | Running (or stopped) **process environment** created from an image |
| **Analogy**   | Class / blueprint                                  | Object / instance                                                  |
| **Disk**      | Shared layers, can be reused by many containers    | Writable thin layer on top of image + optional volumes             |
| **Lifecycle** | Built, tagged, pushed, pulled                      | Created, started, stopped, removed                                 |


**Remember:** **pull an image**, then **run a container**.

---

## 5. Docker vs virtual machine (VM)

**VM:** Full guest OS per machine; hypervisor emulates hardware. **Heavy** start, **large** images/disks.

**Container:** Shares the **host kernel**; isolates processes, filesystem, and network with OS features (namespaces, cgroups). **Faster** start, **smaller** images—**but** still needs a compatible kernel (Linux containers on Linux; Docker Desktop uses a Linux VM on macOS/Windows).

**Typical contrasts:**

- **Size:** Images are often much smaller than VM disks (depends on base image).  
- **Startup:** Containers usually start in seconds; VMs often take minutes.  
- **Portability:** The same image runs on any host with a compatible runtime (not “any CPU architecture” without multi-arch images).

Docker vs virtual machines

---

## 6. Installation

**Official docs:**

- Engine (Linux): [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)  
- Docker Desktop (Mac/Windows, includes GUI): [https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)

**After install, verify:**

```bash
docker version
docker run hello-world
```

**Why `hello-world`?** It checks that the Docker daemon is reachable, that you can pull from a registry, and that a container can start.

---

## 7. Mental model: one-minute workflow

1. **Find or build an image** (Dockerfile / CI / registry).
2. `**docker pull`** if the image is remote and not cached yet.
3. `**docker run`** creates and starts a container from that image.
4. **Ports, volumes, networks** connect the container to the outside world and persistent data.
5. `**docker logs` / `docker exec`** help when something goes wrong.

---

## 8. Core commands

### 8.1 `docker pull` — get an image locally

**Why use it?** You get a specific version (tag) on your machine before `run`. On a slow network, pulling once and then running avoids surprises.

```bash
docker pull nginx:1.25-alpine
docker images
```

**Reading `docker images`:** columns are repository, tag, image ID, size, and created time.

---

### 8.2 `docker run` — create and start a container

**Why use it?** This is the main “run my app” command. Options set networking, names, and whether the container stays in the background.

**Minimal foreground run (logs in the terminal):**

```bash
docker run --rm nginx:1.25-alpine
```

- `**--rm`:** delete the container when it stops (keeps your machine tidy during practice).  
- Stop with `Ctrl+C` in foreground mode.

**Detached run (typical for servers):**

```bash
docker run -d --name web -p 8080:80 nginx:1.25-alpine
```

- `**-d`:** detached (runs in the background).  
- `**--name`:** stable name so you can run `docker stop web`.  
- `**-p 8080:80`:** **host port 8080** → **container port 80** (see section 10).

Open: [http://localhost:8080](http://localhost:8080)

---

### 8.3 `docker ps` — list containers

```bash
docker ps        # running only
docker ps -a     # running + stopped
```

**Why use it?** You need container IDs or names for `stop`, `logs`, and `exec`.

---

### 8.4 `docker start` / `docker stop` — lifecycle without losing the container

**Why use it?** You can keep an existing container (and its writable layer) and pause it. `**docker run` always creates a *new* container.**

```bash
docker stop web
docker start web
```

**Contrast:** `run` = create (new) + start; `start` = start an **existing** container.

---

### 8.5 `docker logs` — stdout/stderr

```bash
docker logs web
docker logs -f web    # follow (like tail -f)
```

**Why use it?** It is usually the fastest way to see crashes, misconfiguration, and stack traces.

---

### 8.6 `docker exec` — run a command *inside* a running container

**Why use it?** The shell inside the container sees **that** filesystem and network—useful to check “what does the app actually see?”

```bash
docker exec -it web sh
# inside: ls /usr/share/nginx/html
# exit with: exit
```

---

### 8.7 `docker inspect` — low-level JSON metadata

```bash
docker inspect web
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
```

**Why use it?** IP addresses, mounts, environment variables, restart policy—when `docker ps` is not enough.

---

### 8.8 `docker cp` — copy files in/out

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf.backup
```

**Why use it?** Quick way to copy configs or logs without rebuilding the image. For production design, volumes are usually a better fit.

---

### 8.9 `docker images` / `docker rmi` — image housekeeping

```bash
docker images
docker rmi nginx:1.25-alpine   # remove local image tag (only if no container uses it)
```

---

## 9. Port mapping (host port ↔ container port)

**Problem:** Inside the container, nginx listens on **port 80**. Your browser talks to **your computer**, not automatically to the container.

**Solution:** **Publish** a host port to a container port:

```bash
docker run -d --name web -p 8080:80 nginx:1.25-alpine
```

- **Left** (`8080`): **host** (what you type in the URL: `localhost:8080`).  
- **Right** (`80`): **container** (what nginx listens on **inside**).

**What happens:** Traffic hits the host → Docker forwards it → container port.

**Common mistake:** The app listens only on `127.0.0.1` inside the container—then port publish will not work from another machine or sometimes from the host as you expect. Bind to `0.0.0.0` when the app must accept external connections.

---

## 10. Hands-on: Nginx step by step (first container)

**Goal:** Serve a static site on localhost and practice pull, run, `ps`, logs, and stop.


| Step | Command / action                                                 | Purpose                                                            |
| ---- | ---------------------------------------------------------------- | ------------------------------------------------------------------ |
| 1    | `docker pull nginx:1.25-alpine`                                  | Use a small, explicit image tag so everyone gets the same version. |
| 2    | `docker run -d --name lab-nginx -p 8080:80 nginx:1.25-alpine`    | Run nginx in the background; map host 8080 → container 80.         |
| 3    | Open [http://localhost:8080](http://localhost:8080)              | Confirm port mapping works.                                        |
| 4    | `docker ps`                                                      | See `lab-nginx` running.                                           |
| 5    | `docker logs lab-nginx`                                          | After refreshing the page, see access/error lines.                 |
| 6    | `docker exec -it lab-nginx sh` then `wget -qO- http://127.0.0.1` | Confirm the app responds **from inside** the container too.        |
| 7    | `docker stop lab-nginx`                                          | Graceful shutdown.                                                 |
| 8    | `docker rm lab-nginx`                                            | Remove the container (skip if you used `--rm` on `run`).           |


**Optional:** Mount a local `html` folder over nginx’s document root (connects to section 12 on volumes).

---

## 11. Networking overview

Docker connects containers using **network drivers**. This course focuses on three: `**bridge`** (default and custom), `**host**`, and `**none**`. Other drivers exist (for example `overlay` in Swarm/Kubernetes), but they are out of scope here.

---

### 11.1 Default `bridge`: reach by IP, not by name; isolated from the host’s network namespace

New containers usually attach to Docker’s **default bridge** (`bridge` in `docker network ls`) unless you pass `--network`.


| Behavior                  | On the **default** bridge                                                                                                                                                                                                                                                                                                                                        |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Container ↔ container** | They can talk using each other’s **container IP** (e.g. `172.17.x.x`). You can discover IPs with `docker inspect <name>`.                                                                                                                                                                                                                                        |
| **DNS by container name** | **No.** The embedded DNS server that maps **names → containers does not apply** on the default bridge the same way it does on a **user-defined** bridge. Using `http://other-container` usually **fails** unless you add manual host entries (below).                                                                                                            |
| **Host vs container**     | Containers are **not** on the host’s LAN as regular NICs. They sit behind Docker’s bridge and **NAT**. From inside a container, “the outside world” is reached via the host’s routing (unless you use `host` mode or special options). So they are **logically isolated** from the host’s network **namespace** even though traffic can go out through the host. |


**Small experiment (default bridge):** Run a container on the default bridge, `curl` it by **IP** (works), then try the same by **container name** (usually fails).

```bash
docker run -d --name nginx-a nginx:1.25-alpine
IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx-a)

docker run --rm curlimages/curl curl -fsS http://${IP}/

# Name-based HTTP usually fails on the default bridge:
docker run --rm curlimages/curl curl -fsS http://nginx-a/ || echo "expected: name not resolved on default bridge"

docker rm -f nginx-a
```

---

### 11.2 Workaround: `--add-host` (manual DNS — not ideal)

You can map a **hostname** to an **IP** when you **run** a container. This fixes “I need a stable name” without a user-defined network, but you must **update** the IP if the other container is recreated (IPs can change). **Prefer a user-defined bridge** for real apps.

**Example:** After you know `nginx-a`’s IP, run a client with a fake hostname:

```bash
docker run -d --name nginx-a nginx:1.25-alpine
IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx-a)

docker run --rm --add-host mynginx:${IP} curlimages/curl curl -fsS http://mynginx/

docker rm -f nginx-a
```

---

### 11.3 Reaching a container from your machine: port mapping (`-p`)

Containers are isolated until you **publish** a port: **host port** → **container port**. Traffic hits your laptop (or server) on `localhost:HOST_PORT`, and Docker forwards it into the container.

```bash
docker run -d --name web -p 8080:80 nginx:1.25-alpine
# Browser or: curl http://localhost:8080
```

This is how you connect **host → container** for HTTP, APIs, databases, etc. It does **not** by itself give you **container name DNS** on the default bridge.

---

### 11.4 Network drivers you will use: `bridge`, `host`, and `none`


| Driver       | What it does                                                                                                                                                                      | Typical use                                          |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `**bridge`** | Containers on a **Linux bridge** with NAT to the outside (unless the network is `--internal`). **User-defined** bridges add **automatic DNS** between containers on that network. | Almost all app containers.                           |
| `**host`**   | The container **shares the host’s network stack** (no separate published ports in the usual sense on Linux). **What you bind on port 80 is the host’s port 80.**                  | Very low latency or legacy apps; **less isolation**. |
| `**none`**   | Only the `loopback` interface inside the container—**no external network**.                                                                                                       | Jobs that must not talk to the network at all.       |


`**host` example (Linux):** `hostname` inside the container should match the **machine’s** hostname, because there is no separate network namespace for the container’s main interfaces.

```bash
hostname   # on the host (Linux)
docker run --rm --network host alpine hostname
# Often the same value on Linux on MacOS  "docker-desktop"
```

**Note (Docker Desktop on Mac/Windows):** `host` refers to the **Linux VM** that runs Docker, not always your Mac/Windows hostname. Check the [Docker Desktop networking docs](https://docs.docker.com/desktop/networking/) for your version.

`**none` example:**

```bash
docker run --rm --network none alpine ip addr
# Typically only lo (loopback); no route to the internet
```

---

### 11.5 User-defined `bridge`: DNS, subnets, and `--internal`

**Why create your own bridge?** You get **DNS**: container `**--name`** becomes a hostname for other containers on **that** network (e.g. `curl http://api`).

**Create a network with a subnet** (optional but useful for planning IP space):

```bash
docker network create --driver bridge --subnet=172.28.0.0/16 app-net
docker network inspect app-net
```

`**--internal`:** Containers on this network can talk **to each other**, but Docker does **not** set up a path to **external** networks (no default outbound to the public internet). Good for **isolated tiers** or demos where outbound must be blocked.

```bash
docker network create --internal isolated-net
docker run --rm --network isolated-net curlimages/curl curl -fsS --max-time 3 https://example.com || echo "expected to fail: no external route"
```

**Useful commands:**

```bash
docker network ls
docker network create mynet
docker run -d --name app --network mynet nginx:1.25-alpine
docker run --rm --network mynet curlimages/curl curl -s http://app
```

On `**mynet**`, the name `**app` resolves** to that container—simple service discovery without Kubernetes.

---

### 11.6 `docker network connect` and `disconnect`

A container can be attached to **more than one** network. You start it on one network, then **connect** it to another (for example, a database on a private network and a front-end network).

**Example:**

```bash
docker network create front-net
docker network create back-net

docker run -d --name web --network front-net nginx:1.25-alpine

# Same container also joins the backend network
docker network connect back-net web

docker inspect web --format '{{json .NetworkSettings.Networks}}' | head -c 500; echo

# Remove only that attachment
docker network disconnect back-net web

docker rm -f web
docker network rm front-net back-net
```

---

### 11.7 Hands-on: nginx and `infoblox/dnstools` on a custom network

**Goal:** Create a **user-defined network**, run **nginx** and a **DNS tools** image on it, then call nginx **by container name** from the tools container (no host port required for container-to-container traffic).

**Why this matters:** On a custom bridge, Docker’s **embedded DNS** resolves `**nginx`** to the nginx container’s IP.

**Steps:**

```bash
# 1) User-defined bridge network
docker network create lab

# 2) Nginx on that network (port 80 inside the network only)
docker run -d --name nginx --network lab nginx:1.25-alpine

# 3) From dnstools: HTTP GET nginx by service name
docker run --rm --network lab infoblox/dnstools curl -fsS http://nginx/
```

You should see the default nginx HTML welcome page in the terminal.

If `curl` is not available in your tag of `infoblox/dnstools`, open a shell and use `wget`:

```bash
docker run --rm -it --network lab infoblox/dnstools sh
# inside the container:
wget -qO- http://nginx/
exit
```

**Optional — prove DNS:**

```bash
docker run --rm --network lab infoblox/dnstools getent hosts nginx
```

**Cleanup:**

```bash
docker rm -f nginx
docker network rm lab
```

---

## 12. Volumes and persistence

**Problem:** The container’s writable layer is **ephemeral** for most workflows—if you delete the container, local data there is gone unless you committed it to a new image (not how you usually run databases).

**Three patterns:**


| Pattern          | When                                                   | Example                       |
| ---------------- | ------------------------------------------------------ | ----------------------------- |
| **Named volume** | Docker-managed, portable                               | `docker volume create pgdata` |
| **Bind mount**   | Dev: edit files on the host, see them in the container | `-v $PWD/app:/app`            |
| **tmpfs**        | Sensitive or scratch data; RAM only                    | `--tmpfs /tmp`                |


```bash
docker volume create demo-data
docker run -d -v demo-data:/data --name vol-demo alpine sleep 3600
docker exec -it vol-demo sh -c 'echo hello > /data/file.txt'
docker rm -f vol-demo
docker run --rm -v demo-data:/data alpine cat /data/file.txt
```

**What this shows:** Data in a **named volume** can **survive** deleting the container.

```bash
docker volume ls
```

---

### 12.1 Example: MySQL with a bind mount (host directory = `$PWD`)

**Goal:** Store MySQL data files in a folder **on your machine** (`$PWD/mysql-data`) so they survive container removal and you can inspect or back up files from the host.

**Why `$PWD`?** The left side of `-v` is a **host path**; using the current directory makes the example easy to copy. On Windows PowerShell, use `${PWD}` or an explicit path.

**Steps:**

```bash
mkdir -p mysql-data

docker run -d \
  --name mysql-bind \
  -e MYSQL_ROOT_PASSWORD=devpassword \
  -v "$PWD/mysql-data:/var/lib/mysql" \
  mysql:8
```

Wait a few seconds for first-time initialization, then check:

```bash
docker logs mysql-bind
ls mysql-data
```

**Cleanup:**

```bash
docker rm -f mysql-bind
# optional: remove data folder
# rm -rf mysql-data
```

Use a **strong** root password in real environments; this is for learning only.

---

### 12.2 Example: MySQL with a Docker **named** volume

**Goal:** Let Docker **manage** storage under its volume directory instead of a folder you choose. Data still survives when you remove the container.

**Difference from bind mount:** You refer to the volume by **name** (`mysql-docker-data`), not by a path on your laptop. Good for “production-like” setups and Compose defaults.

**Steps:**

```bash
docker volume create mysql-docker-data

docker run -d \
  --name mysql-named \
  -e MYSQL_ROOT_PASSWORD=devpassword \
  -v mysql-docker-data:/var/lib/mysql \
  mysql:8
```

Inspect the volume:

```bash
docker volume inspect mysql-docker-data
docker logs mysql-named
```

**Cleanup:**

```bash
docker rm -f mysql-named
docker volume rm mysql-docker-data
```

---

### 12.3 Example: nginx with a custom welcome page (single file bind mount)

**Goal:** Replace the default `index.html` with your own file from the host—**one file** mounted over nginx’s document root file.

**Steps (run from a directory where you want the file):**

```bash
printf '%s\n' '<!DOCTYPE html><html><head><meta charset="utf-8"><title>Welcome</title></head><body><h1>Hello Docker</h1><p>Welcome — this page is mounted from your host.</p></body></html>' > index.html

docker run -d --name nginx-welcome -p 8080:80 \
  -v "$PWD/index.html:/usr/share/nginx/html/index.html:ro" \
  nginx:1.25-alpine
```

- `**:ro**` mounts the file **read-only** inside the container (nginx only needs to read it).
- Open [http://localhost:8080](http://localhost:8080) — you should see **Hello Docker** and the welcome text.
- Edit `index.html` on the host and refresh the browser; changes apply **without** rebuilding an image.

**Cleanup:**

```bash
docker rm -f nginx-welcome
# optional: rm index.html
```

---

## 13. Debugging: try this order

When a container misbehaves, work through these steps:

1. `**docker ps -a**` — Is it running? What was the exit code?
2. `**docker logs <name>**` — Did it crash on startup?
3. `**docker inspect <name>**` — Wrong environment variables? Wrong command?
4. `**docker exec -it <name> sh**` — Can you see the right files? DNS? `curl` to dependencies?
5. **Ports** — Correct `-p`? Is the app listening on `0.0.0.0`?
6. **Volumes** — Is data where you expect? Any permission issues?

---

## 14. Building your own image: Dockerfile

**Why Dockerfiles?** They define a reproducible **build** for **your** image (not only pulling `nginx`). Docker reads the file line by line and creates **layers**; each change that affects the filesystem can become a new layer, which affects build cache and image size.

**Official reference (full syntax and behavior):** [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)

---

### 14.1 Common instructions (what they do)


| Instruction       | Role                                                                                                                                                                                                           |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `**FROM`**        | Sets the **base image** (and starts a **build stage**). Every Dockerfile begins with `FROM` (except optional parser directives). Use tags (`node:20-alpine`) or digests for reproducible builds.               |
| `**WORKDIR`**     | Sets the **current directory** for later `RUN`, `CMD`, `ENTRYPOINT`, `COPY`. Prefer this over `cd` in `RUN`.                                                                                                   |
| `**COPY`**        | Copies files **from your build context** (usually the directory you pass to `docker build`) into the image. Use for source code and config.                                                                    |
| `**ADD`**         | Like `COPY`, but can also fetch URLs and auto-extract local tar archives. For most projects `**COPY` is enough** and easier to reason about.                                                                   |
| `**RUN`**         | Runs a command **while building** the image (install packages, compile, `npm ci`, …). Each `RUN` can create a new layer; chaining commands in one `RUN` reduces layer count when it helps.                     |
| `**ENV`**         | Sets **environment variables** inside the image (available at build time and when the container runs).                                                                                                         |
| `**ARG`**         | **Build-time** variables (e.g. version pins). Not kept in the final image unless referenced by `ENV`. Useful for CI and `docker build --build-arg`.                                                            |
| `**EXPOSE`**      | **Documents** which ports the app uses. It does **not** publish ports by itself—you still use `-p` on `docker run` (or Compose).                                                                               |
| `**CMD`**         | **Default command** when a container starts. If you use `docker run ... myimage node other.js`, that can override `CMD`. Prefer **exec form** `["node", "server.js"]` so signals reach your process correctly. |
| `**ENTRYPOINT`**  | Defines the **main executable**; `CMD` then supplies **default arguments**. Good for wrapping binaries; combine carefully with `docker run` argument rules.                                                    |
| `**USER`**        | Run later `RUN`/`CMD` as a non-root user (better security). Create the user in the image if needed.                                                                                                            |
| `**VOLUME`**      | Declares a mount point for **persistent or shared** data; documents intent and can create an anonymous volume if not overridden at run time.                                                                   |
| `**HEALTHCHECK`** | Tells Docker how to test if the app inside the container is still healthy (optional; common in production Compose/Kubernetes setups).                                                                          |


**Exec form vs shell form:** `RUN ["npm", "ci"]` and `CMD ["node", "app.js"]` avoid a shell wrapper. `RUN npm ci` runs through `/bin/sh -c`. Exec form is usually preferred for `CMD`/`ENTRYPOINT` so the main process is PID 1 and receives signals.

---

### 14.2 Single-stage example (Node)

Good for learning and small apps: one image does install + run.

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Build and run:

```bash
docker build -t myapp:1.0 .
docker run --rm -p 3000:3000 myapp:1.0
```

---

### 14.3 Multi-stage builds (smaller, safer final images)

**Idea:** Use a **first stage** with compilers, build tools, and dev dependencies. Use a **final stage** with only what you need to **run** the app (binary or production assets). The final image does not need Go, gcc, or full `node_modules` with dev packages.

**Benefits:** Smaller images, faster deploys, less attack surface, clearer separation between “build” and “run”.

Each `FROM` starts a new stage. `**AS name`** labels a stage. `**COPY --from=stage`** copies artifacts from an earlier stage into the current one.

---

#### Multi-stage **Go** example

Go compiles to a **single static binary**, which fits multi-stage builds very well: the runtime image can be minimal (e.g. Alpine or distroless).

```dockerfile
# --- build stage: full Go toolchain ---
FROM golang:1.22-alpine AS builder
WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
# static binary, no C linker dependency in final image
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /out/server .

# --- runtime stage: tiny base, no Go SDK ---
FROM alpine:3.19
WORKDIR /app

RUN adduser -D appuser

COPY --from=builder /out/server ./server
USER appuser

EXPOSE 8080
CMD ["./server"]
```

**Why `CGO_ENABLED=0`?** Often produces a fully static binary so you do not need C libraries in the final image (depends on your code and dependencies).

Build and run (adjust port to match your app):

```bash
docker build -t mygoapp:1.0 .
docker run --rm -p 8080:8080 mygoapp:1.0
```

---

#### Multi-stage **Node** example (install → build → production)

Pattern: one stage for **dependencies**, one for **build** (TypeScript, bundler, …), one for **production** with only `NODE_ENV=production` dependencies and built files.

```dockerfile
# --- dependencies ---
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# --- build (compile/bundle) ---
FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# --- production runtime ---
FROM node:20-alpine AS prod
WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --from=build /app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Adjust paths** (`dist/server.js`, `npm run build`) to match your project. If you have **no** build step, you can use two stages only: `deps` + `prod` that copies source and uses `npm ci --omit=dev`.

Build a specific stage (optional):

```bash
docker build --target prod -t mywebapp:1.0 .
```

---

### 14.4 `.dockerignore`

Like `.gitignore` for the Docker **build context**: it keeps `node_modules`, `.git`, secrets, and large unrelated files **out** of the context sent to the daemon. That makes builds **faster** and avoids accidentally copying sensitive files into the image.

---

### 14.5 Quick recap

- Use the **[official Dockerfile reference](https://docs.docker.com/reference/dockerfile/)** for every instruction and edge case.  
- **Single-stage** is simplest; **multi-stage** is standard for production Go, Node, Java, Rust, etc.  
- Prefer `**COPY`** over `**ADD`** unless you need its extra behavior.  
- Use `**CMD**` in exec form and `**EXPOSE**` as documentation; publish ports with `**docker run -p**`.

---

## 15. Docker Compose

**Why Compose?** One file can start **several** containers (app + database + cache) with **networks and volumes**, closer to real projects.

To practice further, use `docker compose up` with a `docker-compose.yml`. The `Docker_The_Hard_Way/` folder in this repository includes examples.

---

## 16. Command cheat sheet (quick reference)


| Command                        | Purpose                                           |
| ------------------------------ | ------------------------------------------------- |
| `docker pull`                  | Download an image                                 |
| `docker images`                | List images                                       |
| `docker run`                   | Create and start a container                      |
| `docker ps` / `docker ps -a`   | List containers                                   |
| `docker stop` / `docker start` | Stop or start an existing container               |
| `docker rm`                    | Remove a container                                |
| `docker rmi`                   | Remove an image                                   |
| `docker logs`                  | Container logs                                    |
| `docker exec`                  | Run a shell or command inside a running container |
| `docker inspect`               | Detailed metadata                                 |
| `docker cp`                    | Copy files between host and container             |
| `docker network` *             | Networks                                          |
| `docker volume `*              | Persistent volumes                                |


---

## 17. Glossary

- **Image:** Immutable template (layers).  
- **Container:** Runnable instance of an image.  
- **Registry:** Image storage (Docker Hub, ECR, …).  
- **Tag:** Label for an image version (`:1.25-alpine`).  
- **Layer:** Reusable filesystem diff in an image.  
- **Volume:** Persistent storage outside the container’s thin writable layer.  
- **Port publish:** Map host port → container port (`-p`).

---

## 18. How to use this material

- Read one section at a time and run the commands on your machine.  
- Do the **Nginx** walkthrough (section 10) and the **volume** example (section 12) for hands-on practice.  
- Keep the **debugging** list (section 13) and **cheat sheet** (section 16) nearby when you work.

---

*Later topics you can explore: security (non-root users, secrets), multi-stage builds, and running images in CI/CD pipelines.*