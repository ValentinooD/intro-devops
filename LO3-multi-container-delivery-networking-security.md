# LO3 — Application Delivery with Containers, Network Architecture, Component Security

Scope: multi-container stacks on the CentOS exam VM with **podman** + podman-compose — pods, user-defined networks, named volumes, secrets, compose files, reverse proxy, `generate kube`/`kube play`. Covers all 23 LO3 practice tasks + a fully worked flagship task (MySQL + WordPress, max security, future scaling) + the security-advantages theory question.

---

## Core concepts (read this first)

**Three ways to wire containers together:**

| Method                                             | Wiring                                     | Discovery                 | Scaling                                                    |
| -------------------------------------------------- | ------------------------------------------ | ------------------------- | ---------------------------------------------------------- |
| **Pod** (`podman pod create`)                      | Containers share ONE network namespace     | `localhost` / `127.0.0.1` | Bad — ports collide inside pod, can't replicate one member |
| **User-defined network** (`podman network create`) | Each container has own IP on a bridge      | DNS by container name     | Good — add more containers to the network                  |
| **Compose** (`compose.yaml`)                       | Declarative: services + networks + volumes | DNS by **service name**   | Good — `--scale`, multiple networks                        |

**Podman DNS rules (exam favourite):**

- The **default `podman` network has DNS DISABLED** (`dns_enabled: false`). Containers on it reach each other only by IP, never by name.
- Every **user-defined network has DNS ENABLED** (`dns_enabled: true`) — containers resolve each other by container name / network alias.
- Containers resolve each other **only if they share at least one network**.
- Check: `podman network inspect <net> | grep dns_enabled`

**Pods = shared namespaces.** A pod has an **infra container** that owns the namespaces. By default containers in a pod share **network, UTS, IPC** namespaces — NOT PID (add with `podman pod create --share net,uts,ipc,pid`). Ports are published **on the pod at creation time** (`podman pod create -p 8080:80`), not per container. Two containers in one pod cannot listen on the same port.

**Volumes:**

- **Named volume** (`podman volume create data`; `-v data:/var/lib/mysql`) — podman-managed, survives `podman rm`, no SELinux problems, best for **data**.
- **Bind mount** (`-v ./conf:/etc/app:Z`) — host directory, editable from host, best for **config/source**. On SELinux systems (CentOS!) you MUST add `:Z` (private label) or `:z` (shared label) or the container gets _Permission denied_.
- Backup/restore: `podman volume export` / `podman volume import`, or a helper alpine container with `tar`.

**Secrets:** `podman secret create name file` (or `printf 'pw' | podman secret create name -`). Attach with `--secret name` → appears as file `/run/secrets/name` inside the container; **not** in `podman inspect`, **not** in `env`, **not** in shell history. Env variant: `--secret name,type=env,target=VAR`.

**Internal networks:** `podman network create --internal net` (compose: `internal: true`) — no gateway, so no traffic to/from the outside world; container-to-container traffic on that network still works. Independently: a DB with **no `-p`/`ports:`** is already unreachable from the host.

**Reverse proxy pattern:** publish only nginx; nginx `proxy_pass`es to app containers by DNS name on a shared network. Enables TLS termination, load balancing across replicas, and keeps apps unpublished.

**Bridge to Kubernetes:** `podman generate kube <pod>` writes Kubernetes YAML from running podman objects; `podman kube play file.yaml` creates them from YAML (`--down` to remove). Same manifests work with `kubectl apply -f` — LO3 → LO4 bridge.

---

## Command cheatsheet

```bash
# Networks
podman network create appnet                      # user-defined, DNS enabled
podman network create --internal dbnet            # no external connectivity
podman network inspect appnet                     # check dns_enabled, subnet, containers
podman network connect appnet mycontainer        # attach running container to a 2nd network

# Pods
podman pod create --name mypod -p 8080:80         # ports are set on the POD
podman run -d --pod mypod --name app <image>      # join a container to the pod
podman pod inspect mypod                          # SharedNamespaces, infra container
podman pod ps ; podman pod rm -f mypod

# Volumes
podman volume create dbdata
podman volume export dbdata --output dbdata.tar   # backup
podman volume import dbdata dbdata.tar            # restore

# Secrets
printf 'S3cret' | podman secret create dbpass -   # create from stdin
podman secret ls ; podman secret rm dbpass
podman run -d --secret dbpass <image>             # file at /run/secrets/dbpass

# Compose (podman compose == podman-compose)
podman compose up -d                               # create nets/volumes, start services
podman compose ps ; podman compose logs -f app    # observe
podman compose up -d --scale app=3                # replicas
podman compose down                                # remove containers (volumes stay)
podman compose down -v                             # ALSO remove volumes

# Kube bridge
podman generate kube mypod -s > mypod.yaml         # -s adds a Service
podman kube play mypod.yaml ; podman kube play --down mypod.yaml
```

---

## Practice tasks 3.1–3.23

### {3.1} Pod with app + DB talking over localhost

> Q: Create a pod with `podman pod create` (publishing a port), add an app container and a database container to it, and show they reach each other over localhost.

**DO**

```bash
podman pod create --name wp-pod -p 8080:80

podman run -d --pod wp-pod --name pod-db \
  -e MYSQL_ROOT_PASSWORD=r00tpw -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser -e MYSQL_PASSWORD=wppw \
  docker.io/library/mysql:8.0

podman run -d --pod wp-pod --name pod-app \
  -e WORDPRESS_DB_HOST=127.0.0.1 \
  -e WORDPRESS_DB_USER=wpuser -e WORDPRESS_DB_PASSWORD=wppw \
  -e WORDPRESS_DB_NAME=wordpress \
  docker.io/library/wordpress:latest
```

**VERIFY**

```bash
podman pod ps                     # 3 containers (infra + db + app), port 8080->80
curl -s http://localhost:8080 | head        # WordPress answers (it connected to DB on 127.0.0.1)
podman run --rm --pod wp-pod docker.io/library/mysql:8.0 \
  mysql -h127.0.0.1 -uroot -pr00tpw -e 'SELECT 1'   # 3rd container reaches DB on localhost
```

**WHY** All containers in a pod share the infra container's network namespace → one IP, one `localhost`. That is why `WORDPRESS_DB_HOST=127.0.0.1` works and why the port was published on the pod, not on a container.

### {3.2} User-defined network + DNS by name

> Q: Create a user-defined network, run an app and a postgres container on it, prove the app resolves the DB by container name.

**DO**

```bash
podman network create appnet
podman run -d --name pgdb --network appnet \
  -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman run -d --name app --network appnet docker.io/library/alpine sleep infinity
```

**VERIFY**

```bash
podman network inspect appnet | grep dns_enabled   # "dns_enabled": true
podman exec app ping -c2 pgdb                       # resolves by NAME
podman exec app getent hosts pgdb                   # shows pgdb's IP
```

**WHY** User-defined networks run a DNS resolver (aardvark-dns) that maps container names/aliases to IPs. The default `podman` network has `dns_enabled: false` → this would fail there.

### {3.3} Two-tier stack that is NOT WordPress/Drupal/Joomla (Gitea + PostgreSQL)

> Q: Deploy a two-tier stack of your choice (not Drupal/Joomla/WordPress) and complete first-run setup.

**DO**

```bash
podman network create gitea-net
podman volume create gitea-dbdata && podman volume create gitea-appdata

podman run -d --name gitea-db --network gitea-net \
  -e POSTGRES_USER=gitea -e POSTGRES_PASSWORD=gitea -e POSTGRES_DB=gitea \
  -v gitea-dbdata:/var/lib/postgresql/data \
  docker.io/library/postgres:16

podman run -d --name gitea --network gitea-net -p 3000:3000 \
  -v gitea-appdata:/data \
  docker.io/gitea/gitea:latest
```

First-run setup: browser → `http://localhost:3000` → Initial Configuration → Database Type **PostgreSQL**, Host `gitea-db:5432`, User `gitea`, Password `gitea`, DB name `gitea` → create admin account → **screenshot**.

**VERIFY** `podman ps` shows both Up; after install, log in to Gitea and create a repo.
**WHY** App resolves `gitea-db` via network DNS; DB has no published port (host can't touch it); named volumes persist both tiers.

### {3.4} Named volume survives container re-creation

> Q: Persist the database's data in a named volume so it survives `podman rm` + re-creation; prove data is still there.

**DO**

```bash
podman volume create dbdata
podman run -d --name db -e MYSQL_ROOT_PASSWORD=r00tpw \
  -v dbdata:/var/lib/mysql docker.io/library/mysql:8.0
sleep 30
podman exec db mysql -uroot -pr00tpw \
  -e "CREATE DATABASE exam; CREATE TABLE exam.t (msg VARCHAR(50)); INSERT INTO exam.t VALUES ('survives');"

podman rm -f db          # destroy the container
podman run -d --name db -e MYSQL_ROOT_PASSWORD=r00tpw \
  -v dbdata:/var/lib/mysql docker.io/library/mysql:8.0
sleep 30
```

**VERIFY**

```bash
podman exec db mysql -uroot -pr00tpw -e "SELECT * FROM exam.t;"   # -> survives
podman volume inspect dbdata                                      # Mountpoint on host
```

**WHY** The container's copy-on-write layer dies with `podman rm`; a named volume lives in podman's storage independent of any container lifecycle.

### {3.5} Podman secret instead of --env

> Q: Use a podman secret (`podman secret create` + `--secret`) to pass the DB password instead of `--env`; explain the security benefit.

**DO**

```bash
printf 'S3cretR00t!' | podman secret create mysql_root -
podman run -d --name sec-db \
  --secret mysql_root \
  -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql_root \
  docker.io/library/mysql:8.0
# env-var style if the image has no *_FILE support:
#   --secret mysql_root,type=env,target=MYSQL_ROOT_PASSWORD
```

**VERIFY**

```bash
podman exec sec-db cat /run/secrets/mysql_root     # secret is a file in the container
podman exec sec-db env | grep -i password          # only the *_FILE path, no plaintext
podman inspect sec-db | grep -i secretr00t         # nothing — not in inspect
```

**WHY** `--env PASSWORD=x` leaks via `podman inspect`, `podman exec env`, shell history, and process listings. A secret is mounted at runtime only (`/run/secrets/<name>`, tmpfs), never baked into the image or visible in inspect output.

### {3.6} compose.yaml for app + database

> Q: Write a compose file for an app + database, bring it up with `podman compose up -d`, show both services running.

**DO** — `compose.yaml` (Ghost blog + MySQL):

```yaml
services:
  db:
    image: docker.io/library/mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: r00tpw
      MYSQL_DATABASE: ghost
      MYSQL_USER: ghost
      MYSQL_PASSWORD: ghostpw
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - ghost-net

  app:
    image: docker.io/library/ghost:5
    environment:
      database__client: mysql
      database__connection__host: db
      database__connection__user: ghost
      database__connection__password: ghostpw
      database__connection__database: ghost
      url: http://localhost:8080
    ports:
      - "8080:2368"
    depends_on:
      - db
    networks:
      - ghost-net

volumes:
  db-data: {}

networks:
  ghost-net: {}
```

```bash
podman compose up -d
```

**VERIFY** `podman compose ps` → both Up; `curl -s http://localhost:8080 | head` → Ghost HTML.
**WHY** Compose declares services/networks/volumes in one file; podman-compose translates it into `podman network create` / `podman run` commands. App reaches DB via service name `db` (compose networks are DNS-enabled).

### {3.7} DB on internal network, no published port

> Q: Keep the database on an internal network with no published port and expose only the app; prove the DB isn't reachable from the host.

**DO** — change the networks of 3.6:

```yaml
services:
  db:
    # ... as before, NO ports: section
    networks: [backend]
  app:
    # ... as before
    ports: ["8080:2368"]
    networks: [backend, frontend]

networks:
  backend:
    internal: true # no gateway -> no traffic in/out of the host/network
  frontend: {}
```

**VERIFY**

```bash
podman compose up -d
nc -zv 127.0.0.1 3306                 # Connection refused — DB not on host
podman port <db-container>            # no output, nothing published
podman exec <app-container> sh -c 'nc -zv db 3306'   # open — app still reaches it
curl -s localhost:8080 | head         # app works
```

**WHY** No `ports:` means no host mapping at all; `internal: true` additionally removes the gateway so the DB cannot even initiate traffic to the outside. Attack surface = only the app port.

### {3.8} healthcheck + depends_on: service_healthy

> Q: Add a compose healthcheck plus `depends_on: condition: service_healthy` so the app waits for the DB; demonstrate the ordering.

**DO**

```yaml
services:
  db:
    image: docker.io/library/mysql:8.0
    environment: { MYSQL_ROOT_PASSWORD: r00tpw, MYSQL_DATABASE: appdb }
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 20s
  app:
    image: docker.io/library/alpine
    command: sh -c "echo DB is ready, starting app; sleep infinity"
    depends_on:
      db:
        condition: service_healthy
```

**VERIFY**

```bash
podman compose up -d
podman ps        # db shows "(starting)" then "(healthy)"; app is created only after healthy
podman inspect --format '{{.State.Health.Status}}' <db-container>   # healthy
```

**WHY** Plain `depends_on` only orders **starts**; MySQL takes ~20 s to accept connections, so apps crash on first connect. `condition: service_healthy` blocks the app until the healthcheck passes.

### {3.9} env_file for credentials

> Q: Use an `env_file:` in compose for the DB credentials instead of inline values.

**DO** — `db.env`:

```bash
MYSQL_ROOT_PASSWORD=r00tpw
MYSQL_DATABASE=appdb
MYSQL_USER=appuser
MYSQL_PASSWORD=apppw
```

```yaml
services:
  db:
    image: docker.io/library/mysql:8.0
    env_file:
      - db.env
```

```bash
chmod 600 db.env && podman compose up -d
```

**VERIFY** `podman exec <db> env | grep MYSQL` — vars are set.
**WHY** Credentials leave the compose file (which you commit/screenshot); one env file per environment; restrict with file permissions and `.gitignore`. (Still weaker than secrets — values do end up in the container env.)

### {3.10} Scale a compose service

> Q: Scale one compose service to multiple replicas and explain how requests are distributed.

**DO** — the scaled service must have **no `container_name`** and **no fixed host port**:

```yaml
services:
  app:
    image: docker.io/library/httpd:2.4 # no ports:, no container_name:
```

```bash
podman compose up -d --scale app=3
```

**VERIFY** `podman compose ps` → `<dir>_app_1`, `_app_2`, `_app_3` all Up.
**WHY** Replicas share the DNS alias `app`; podman's DNS returns the replicas' records so connections spread crudely (DNS round-robin). Real distribution needs a **reverse proxy/load balancer** (nginx/HAProxy) in front — which is also why a fixed `container_name` or host port would break scaling (name/port collisions).

### {3.11} Bind mount for config + named volume for data in one container

> Q: Mount a bind mount for app configuration and a named volume for data in the same container; explain why you'd use each.

**DO**

```bash
mkdir -p ./conf && printf 'server { listen 80; root /data; autoindex on; }\n' > ./conf/site.conf
podman volume create web-data
podman run -d --name web -p 8080:80 \
  -v ./conf/site.conf:/etc/nginx/conf.d/default.conf:ro,Z \
  -v web-data:/data \
  docker.io/library/nginx
```

**VERIFY** `podman exec web cat /etc/nginx/conf.d/default.conf`; `podman exec web touch /data/x` then `podman volume inspect web-data`.
**WHY** Bind mount = config you edit on the host / keep in git; needs `:Z` for SELinux, `ro` for safety. Named volume = data podman manages; survives container replacement, portable via export/import, no SELinux labeling issues.

### {3.12} Back up + restore a named volume with a helper container

> Q: Back up a named volume to a tarball using a helper alpine/busybox container, then restore into a fresh volume.

**DO**

```bash
mkdir -p ./backup
podman run --rm \
  -v dbdata:/data:ro \
  -v ./backup:/backup:Z \
  docker.io/library/alpine tar czf /backup/dbdata.tar.gz -C /data .

podman volume create dbdata-restored
podman run --rm \
  -v dbdata-restored:/data \
  -v ./backup:/backup:Z \
  docker.io/library/alpine tar xzf /backup/dbdata.tar.gz -C /data
```

(Shortcut without helper: `podman volume export dbdata --output dbdata.tar` / `podman volume import dbdata-restored dbdata.tar`.)
**VERIFY** `podman run --rm -v dbdata-restored:/data docker.io/library/alpine ls -l /data` — same files.
**WHY** Volumes aren't ordinary directories you should tar from the host (rootless storage lives under `~/.local/share/containers`); a throwaway container mounts the volume and streams it out portably. `ro` on the source prevents accidental writes during backup.

### {3.13} nginx reverse proxy in front of an app

> Q: Put an nginx reverse-proxy container in front of an app container on a shared network and route host traffic through nginx.

**DO**

```bash
podman network create proxy-net
podman run -d --name app --network proxy-net docker.io/library/httpd:2.4   # listens on 80, NOT published

cat > ./proxy.conf <<'EOF'
server {
    listen 80;
    location / {
        proxy_pass http://app:80;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
EOF

podman run -d --name proxy --network proxy-net -p 8080:80 \
  -v ./proxy.conf:/etc/nginx/conf.d/default.conf:ro,Z \
  docker.io/library/nginx
```

**VERIFY** `curl -s http://localhost:8080` → `It works!` (httpd's page, served through nginx). `podman port app` → nothing.
**WHY** Only the proxy is exposed; nginx resolves `app` via network DNS. This is the pattern for TLS termination, path routing, and load-balancing scaled replicas (add `upstream` with several names).

### {3.14} Database admin container (adminer)

> Q: Add a database-admin container to the network and use it to inspect the database.

**DO** (reuses `appnet`/MySQL from earlier; for postgres use the same idea or pgadmin)

```bash
podman run -d --name adminer --network appnet -p 8081:8080 docker.io/library/adminer
```

Browser → `http://localhost:8081` → System `MySQL`, Server `db` (the DB **container name**, not localhost!), user/password from the DB container → browse tables.
**VERIFY** Adminer login succeeds and lists the schema.
**WHY** Adminer runs in its own container, so `localhost` would mean _itself_ — it must use the DB's DNS name on the shared network. Classic exam trap.

### {3.15} podman generate kube

> Q: Generate Kubernetes YAML from a running pod and explain how it bridges LO3 to LO4.

**DO**

```bash
podman generate kube wp-pod -s > wp-pod.yaml    # -s also generates a Service
cat wp-pod.yaml                                  # apiVersion: v1, kind: Pod (+ Service)
```

**VERIFY** File contains `kind: Pod` with both containers, images, env, ports; `kind: Service` with the published port.
**WHY** Podman speaks Kubernetes YAML: what you built imperatively with podman becomes a declarative manifest you can `kubectl apply -f` on a real cluster — develop locally with podman (LO3), deploy to Kubernetes (LO4).

### {3.16} podman kube play

> Q: Run a pod from Kubernetes YAML with `podman kube play`, then tear it down.

**DO**

```bash
podman pod rm -f wp-pod              # remove the original first (name clash)
podman kube play wp-pod.yaml
```

**VERIFY** `podman pod ps` + `podman ps --pod` → pod and containers recreated from YAML; app answers on its port.
**DOWN**

```bash
podman kube play --down wp-pod.yaml  # deletes the pod + containers it created
```

**WHY** `kube play` is the inverse of `generate kube` — declarative, repeatable deployment on a single host, same file a cluster would consume.

### {3.17} CPU/memory limits per compose service

> Q: Set per-service CPU/memory limits in a compose file and verify with `podman stats`.

**DO**

```yaml
services:
  app:
    image: docker.io/library/httpd:2.4
    mem_limit: 256m
    cpus: 0.5
```

```bash
podman compose up -d
```

**VERIFY** `podman stats --no-stream` → MEM LIMIT column shows `256MiB`; `podman inspect <app> --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'` → `268435456 500000000`.
**WHY** Limits are enforced by cgroups: exceeding memory → OOM-kill of the container process; `cpus: 0.5` throttles to half a core. Protects the host and co-tenant containers (noisy neighbour).

### {3.18} Pod namespaces: shared vs not

> Q: Inspect a pod and show which namespaces the containers share (network, IPC) and which they don't (PID by default).

**DO**

```bash
podman pod inspect wp-pod --format '{{.SharedNamespaces}}'
# -> [ipc net uts]   (no pid)
podman pod create --name pidpod --share net,uts,ipc,pid   # opt in to shared PID
```

**VERIFY** In `wp-pod`, `podman exec pod-app ps aux` shows only the app's processes (own PID ns). In `pidpod`, containers see each other's processes.
**WHY** Shared **net** → one IP/localhost; shared **IPC** → shared memory segments; shared **UTS** → same hostname. PID stays private by default for isolation — one container can't inspect/kill another's processes unless you explicitly `--share pid`.

### {3.19} DNS fails across different networks — then fix it

> Q: Demonstrate name-based discovery failing when two containers are on different networks, then fix by attaching them to a common network.

**DO**

```bash
podman network create net-a && podman network create net-b
podman run -d --name c1 --network net-a docker.io/library/alpine sleep infinity
podman run -d --name c2 --network net-b docker.io/library/alpine sleep infinity
podman exec c1 ping -c1 c2          # FAILS: bad address 'c2'
podman network connect net-a c2     # attach running c2 to net-a as well
podman exec c1 ping -c1 c2          # SUCCEEDS
```

**VERIFY** `podman inspect c2 --format '{{.NetworkSettings.Networks}}'` → now lists both networks.
**WHY** Podman DNS only resolves names of containers on a **shared** network — networks are isolation boundaries. `podman network connect` hot-plugs a second interface without restarting the container.

### {3.20} One container on two networks (frontend + backend)

> Q: Attach a single container to two networks and explain the segmentation benefit.

**DO**

```bash
podman network create frontend && podman network create backend
podman run -d --name db  --network backend -e MYSQL_ROOT_PASSWORD=r00tpw docker.io/library/mysql:8.0
podman run -d --name api --network frontend,backend docker.io/library/httpd:2.4
podman run -d --name ui  --network frontend -p 8080:80 docker.io/library/nginx
```

**VERIFY** `podman exec ui getent hosts api` → resolves; `podman exec ui getent hosts db` → FAILS (ui has no route/DNS to backend); `podman exec api getent hosts db` → resolves.
**WHY** The api is the only bridge between tiers. If the ui is compromised, the DB is unreachable from it — network segmentation = smaller blast radius, same idea as DMZ zoning.

### {3.21} Observe a stack with compose logs / ps

> Q: Use `podman compose logs` and `podman compose ps` to observe and troubleshoot a multi-service stack.

**DO**

```bash
podman compose ps                # state + ports of every service
podman compose logs db           # one service's logs
podman compose logs -f           # follow all services, interleaved with prefixes
```

**VERIFY** A failing DB shows e.g. `MYSQL_ROOT_PASSWORD not set` in `logs db`; `ps` shows `Exited (1)`.
**WHY** First-response triage order: `compose ps` (what died?) → `compose logs <svc>` (why?) → fix compose file → `podman compose down && podman compose up -d`. Container names in the output follow `<dir>_<service>_<n>`.

### {3.22} Shared named volume across two replicas + the risk

> Q: Share configuration across two replicas of the same app via a shared named volume; explain a risk of shared read-write storage.

**DO**

```bash
podman volume create shared-html
podman run -d --name web1 -p 8081:80 -v shared-html:/usr/share/nginx/html docker.io/library/nginx
podman run -d --name web2 -p 8082:80 -v shared-html:/usr/share/nginx/html docker.io/library/nginx
podman exec web1 sh -c 'echo "hello from shared volume" > /usr/share/nginx/html/index.html'
```

**VERIFY** `curl -s localhost:8081` and `curl -s localhost:8082` → identical content written once.
**WHY** One volume, many mounts → instant config consistency. **Risk:** concurrent read-write with no locking — two writers can interleave/corrupt files (never share one data directory between two DB instances: guaranteed corruption). Prefer read-only mounts (`:ro`) for all but one writer.

### {3.23} compose down vs down --volumes

> Q: Tear down a stack with `podman compose down`; what happens to named volumes and how does `--volumes` change that?

**DO**

```bash
podman compose down        # stops + removes containers (and the default network)
podman volume ls           # named volumes STILL THERE — data preserved
podman compose up -d       # comes back with all data intact
podman compose down -v     # -v / --volumes: ALSO removes the volumes declared in the file
podman volume ls           # gone — data destroyed
```

**VERIFY** After plain `down`+`up`, the DB still has its tables; after `down -v`, the app re-runs first-time setup.
**WHY** Volumes are deliberately decoupled from container lifecycle so `down` is safe. `-v` is the intentional "wipe the environment" switch — never use it on anything you care about without a backup (3.12).

---

## THE EXAM QUESTION — MySQL + WordPress, maximum security, future scaling (8 pts)

Design decisions (say these out loud in your answer):

1. **User-defined network, NOT a pod** — pods share one netns: WordPress replicas would collide on port 80 and you can't scale one member of a pod. A network lets you add `wordpress2`, `wordpress3` + a reverse proxy later, or `podman compose up --scale`.
2. **DB port NOT published** — MySQL reachable only by containers on `wp-net`, invisible to the host/outside.
3. **podman secrets** for both passwords (never `--env` with plaintext; `*_FILE` variables read them from `/run/secrets/`).
4. **Least privilege**: WordPress uses a dedicated `wpuser`, not root.
5. **Named volumes** for `/var/lib/mysql` and `/var/www/html` — data survives re-creation.
6. **Publish only WordPress on 8080**; rootless podman (no root daemon); pin image tags.

### Version A — pure podman

```bash
# 1. network (DNS enabled, isolated from other containers)
podman network create wp-net

# 2. secrets
printf 'R00t-Sup3rSecret!' | podman secret create mysql_root -
printf 'Wp-Us3rSecret!'    | podman secret create mysql_wp -

# 3. volumes
podman volume create wp-db
podman volume create wp-html

# 4. database — no -p, passwords via secrets, own volume
podman run -d --name wp-mysql --network wp-net \
  --secret mysql_root --secret mysql_wp \
  -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql_root \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD_FILE=/run/secrets/mysql_wp \
  -v wp-db:/var/lib/mysql \
  --restart=always \
  docker.io/library/mysql:8.0

# 5. wordpress — only published component, connects by DNS name
podman run -d --name wp-app --network wp-net -p 8080:80 \
  --secret mysql_wp \
  -e WORDPRESS_DB_HOST=wp-mysql \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD_FILE=/run/secrets/mysql_wp \
  -e WORDPRESS_DB_NAME=wordpress \
  -v wp-html:/var/www/html \
  --restart=always \
  docker.io/library/wordpress:latest
```

**VERIFY**

```bash
podman ps                                   # both Up; only wp-app has 0.0.0.0:8080->80
curl -s http://localhost:8080 | head        # WordPress install page -> finish in browser, screenshot
nc -zv 127.0.0.1 3306                       # refused: DB not reachable from host
podman exec wp-app env | grep -i pass       # only the *_FILE path, no plaintext password
```

### Version B — compose.yaml (same stack, declarative)

```yaml
services:
  db:
    image: docker.io/library/mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_wp
    secrets: [mysql_root, mysql_wp]
    volumes:
      - wp-db:/var/lib/mysql
    networks: [backend] # internal net, NO ports:
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 20s

  wordpress:
    image: docker.io/library/wordpress:latest
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/mysql_wp
      WORDPRESS_DB_NAME: wordpress
    secrets: [mysql_wp]
    volumes:
      - wp-html:/var/www/html
    ports:
      - "8080:80" # ONLY published port in the stack
    networks: [backend, frontend]
    depends_on:
      db:
        condition: service_healthy
    restart: always

volumes:
  wp-db: {}
  wp-html: {}

networks:
  backend:
    internal: true # DB tier cannot reach/be reached from outside
  frontend: {}

secrets:
  mysql_root:
    external: true # created with: printf '...' | podman secret create mysql_root -
  mysql_wp:
    external: true
```

```bash
podman compose up -d
```

(If `external: true` secrets misbehave with your podman-compose version, fall back to `file: ./secrets/mysql_root.txt` with `chmod 600` files — still keeps passwords out of the YAML.)

**Future scaling argument (write this):** because the tiers are joined by a **network** (not a pod), scaling = add replicas: `podman compose up -d --scale wordpress=3` (drop the fixed host port, put the nginx reverse proxy of 3.13 in front on `frontend` and publish only nginx), all replicas share the `wp-html` volume and the single DB. Beyond one host, `podman generate kube` exports the same stack toward Kubernetes.

### Theory (4 pts): "Evaluate advantages of container delivery in terms of security" — model answer

- **Isolation**: each component runs in its own namespaces/cgroups — a compromised WordPress cannot see the DB's processes, filesystem, or memory; kernel-enforced boundaries.
- **Minimal attack surface**: images ship only the app + its libraries (no SSH, no extra daemons); minimal/distroless bases mean fewer packages → fewer CVEs.
- **Immutability**: containers are recreated from images, not patched in place; drift and persistent backdoors are eliminated; a compromised container is discarded and redeployed clean.
- **Fast patching**: rebuild the image on a fixed base, redeploy in seconds — patch once, everywhere.
- **Image scanning & signing**: registries scan layers for CVEs; digests/signatures guarantee you run exactly what was built (supply-chain integrity).
- **Secrets management**: podman secrets keep credentials out of images, env vars, inspect output, and shell history.
- **Rootless & daemonless podman**: containers run under an unprivileged user, no root daemon to hijack — container escape lands as a normal user.
- **SELinux/seccomp/capabilities**: mandatory access control confines each container; capabilities are dropped by default.
- **Network segmentation**: user-defined/internal networks expose only what must be public; the DB is simply not reachable.
- **Resource limits**: cgroup CPU/memory caps contain DoS/noisy-neighbour effects.

---

## Traps & gotchas

- **Default `podman` network has NO DNS** — `ping othercontainer` fails there. Always `podman network create` first. Verify with `podman network inspect <net> | grep dns_enabled`.
- **Pods share localhost — ports collide.** Two containers in one pod can't both listen on 80; publish ports on `podman pod create -p ...`, not on `podman run`. Adding `-p` to a container joining a pod is an error.
- **In a pod, DB host = `127.0.0.1`; on a network, DB host = container/service name.** Mixing these up is the #1 broken-stack cause.
- **`podman compose down -v` deletes named volumes** (data gone). Plain `down` keeps them.
- **Bind mounts on CentOS need `:Z`** (or `:z`) or SELinux gives _Permission denied_ / empty dirs. Named volumes never need it.
- **`internal: true` blocks external traffic, not container-to-container** — containers on the internal net still talk to each other; it is NOT a firewall between them.
- **Secrets are invisible by design**: not in `podman inspect`, not in `env` (file mode) — look in `/run/secrets/<name>`. If the image needs an env var, use `type=env,target=VAR` or the image's `*_FILE` convention.
- **`depends_on` alone ≠ ready** — it orders starts only; without `condition: service_healthy` + a healthcheck, the app races a still-initializing MySQL and crashes.
- **Scaling breaks with `container_name:` or fixed host `ports:`** on the scaled service — names/ports would collide; use a reverse proxy for ingress instead.
- **From inside any container, `localhost` is that container** (unless in a pod) — adminer/pgadmin must use the DB's DNS name, never `localhost`.
