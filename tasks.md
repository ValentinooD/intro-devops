# intro-to-devops-final-notes

### Log in to a secure container image registry:
podman login registry.redhat.io

### Log out 
podman logout --all 

### Download container image to my host 
podman pull registry.access.redhat.com/ubi9/ubi-minimal:9.5 

### List images
podman images 

### Run container image (to check if it's working)
podman run registry.access.redhat.com/ubi9/ubi-minimal:9.5 echo 'hello red hat'

### Run container image and automatically remove after running 
podman run --rm registry.access.redhat.com/ubi9/ubi-minimal:9.5 echo 'Red Hat'

### Run container image in the background (detached mode)
podman run -d registry.access.redhat.com/ubi9/ubi-minimal:9.5

### Assign name to container
podman run --name my-container-name my-image

### Map public port on my host machine (host first : public port)
podman run -p 8081:8080 registry.access.redhat.com/ubi9/ubi-minimal:9.5

### Map public port on my host machine to a specifi IP
podman run -p 127.0.0.1:8082:8080 registry.access.redhat.com/ubi9/ubi-minimal:9.5

### List port mapping for a container
podman port instance-name

### List ports for all containers
podman port --all

### Add environmental variable
podman run --env KEY='Value' IMAGE

### List all currently active and running containers on the host
podman ps

### List all containers on the system (even if they're not running)
podman ps --all

### Stop a running container 
podman stop my-container

### Stop all running containers
podman stop --all

### Force kill active container
podman kill my-container

### Remove a container
podman rm my-container

### Forcefully remove container
podman rm my-container --force

### Remoe all non-running containers
podman rm --all

### Start process in container
podman exec my-container

### Make it interactive with terminal
podman exec -it my-container /bin/bash

### Copy files in and out of conatiners
podman cp /host/path/file.txt my-container:/container/path/

### Pause/unpause container
podman pause my-container, podman unpause my-container

### Inspect container
podman inspect my-container

### Logging (--tail 20 for num of lines, -f for live log, --since 10m for specific time)
podman logs --tail 20 -f --since 10m my-container

### Create podman network
podman network create my-network

### Block traffic from other networks
podman network create -o isolate=TRUE my-network

### List existing networks
podman network ls 

### Check config data for the network
podman network inspect my-network

### Remove netowrk
podman network rm my-network

### Remove all networks not being used by containers
podman network prune 

### Connect running container to existing network
podman network connect 

### Disconnect a container from a network
podman network disconnect 

### Get IP addrees of container within network
podman inspect app-name --format ‘{{.NetworkSettings.IPAddress}}’ 

### Build an image
podman build --file CONTAINERFILE --tag IMAGETAG

### Push an image
podman push IMAGETAG quay.io/QUAYUSERNAME/IMAGENAME:TAG 

### Inspect image
podman image push IMAGE_REGISTRY_URL

### Remove image
podman image rm REGISTRY/NAMESPACE/IMAGE_NAME:TAG

### Creat volume 
podman volume create VOLUME_NAME

### Import volume
podman volume import VOLUME_NAME ARCHIVE_NAME

### To export a volume
podman volume export VOLUME_NAME --output ARCHIVE_NAME

### Compose file

name: my-app-stack

services:
  - web-service:
    - container_name: frontend-web
    - image: docker.io/library/httpd:latest
    - ports:
      - "8080:80"
    - networks:
      - app-net
    - environment:
      - ENV_MODE: production

  - db-service:
    - container_name: backend-db
    - image: registry.access.redhat.com/rhel9/postgresql-13:1
    - environment:
      - POSTGRESQL_USER: admin
      - POSTGRESQL_PASSWORD: redhat
    - volumes:
      - db-data:/var/lib/pgsql/data:Z
    - networks:
      - app-net

volumes:
  - db-data: {}

networks:
  - app-net: {}

### Execute compose file
podman-compose up

### Stop and remove containers defined as services 
podman-compose down

### Stop the containers that are defined in the file 
podman-compose stop

### Create a podman secret
echo “secret texrt > SECRET_FILENAME
podman secret create SECRET_NAME SECRET_FILENAME 

### Specify driver
podman secret create SECRET_NAME --driver pass 

### Delete secret
podman secret rm SECRET_NAME

### Pass secret as environmental variable
podman run -it --secret dbsecret,type=env,target=DB_SECRET \
--name myapp registry.access.redhat.com/ubi8/ubi /bin/bash


## PRACTICE EXAM ANSWERS (LO1)
### 1. Run an httpd container detached, publish container port 80 to host port 8081, give it a custom --name and a custom --hostname, and confirm both with podman inspect
podman run -d --name web-server --hostname my-httpd-host -p 8081:80 docker.io/library/httpd:latest
podman inspect web-server

### 2. Run a container with --restart=always, kill its main process, and prove podman restarts it (check podman ps and the restart behavior)
podman run -d --name my-container --restart=always docker.io/library/alpine:latest
podman kill my-container
podman ps --all 

### 3. Run a container with --memory=256m --cpus=0.5 and verify the limits are applied using podman stats and podman inspect
podman run -d --name my-container --memory=256m --cpus=0.5 docker.io/library/alpine:latest
podman stats
podman inspect my-container

### 4. Start one container with --env KEY=VALUE and another with --env-file, then show the environment differences with podman exec <c> env
podman run -d --name my-container-1 --env KEY=VALUE my-image
podman run -d --name my-container-2 --env-file ./file.env my-image
podman exec my-container-1 env
podman exec my-container-2 env

### 5. Create a user-defined bridge network with podman network create, run a container attached to it, and inspect the network to find the container's IP
podman network create my-network
podman run -d --name my-container --net my-network my-image
podman network inspect my-network

### 6. Pause and unpause a running container and explain what state its processes are in while paused
podman pause my-container
podman unpause my-container

### 7. Rename an existing container with podman rename and confirm the change.
podman rename old-container-name new-container-name
podman ps --all

### 8. Use podman logs with --tail, --since, and -f, and explain what each flag does.
podman logs --tail 20 my-container
podman logs --since 10m my-container
podman logs -f my-container

### 9. Extract a single field (e.g. the IP address or restart count) using podman inspect --format '{{ ... }}' Gotemplate syntax
podman inspect --format '{{.NetworkSettings.IPAddress}}' my-container

### 10. Copy a file from the host into a running container and back out again with podman cp.
podman cp ./local-file.txt my-container:/tmp/local-file.txt
podman cp my-container:/tmp/local-file.txt ./restored-file.txt

### 11. Exec an interactive shell into a running container, install a package, and explain whether that change survives the container being recreated.
podman exec -it my-container /bin/bash
apk add curl

### 12. Use podman top and podman stats to show the processes and live resource usage of a container.
podmam top my-container
podman stats my-container

### 13. Run a container with --rm, and explain when auto-removal is useful and what you lose by using it.
podman run --rm my-container

### 14. Bind-mount a host directory into a container read-only (-v hostdir:/path:ro) and prove that writes from inside are rejected.
podman run -it --rm -v hostdir:/mnt/data:ro my-image /bin/bash
touch /mnt/data/test.txt

### 14. Bind-mount a host directory into a container read-only (-v hostdir:/path:ro) and prove that writes from inside are rejected.
podman run -it --rm -v hostdir:/mnt/data:ro my-image /bin/bash
touch /mnt/data/test.txt

### 15. Add custom --labels to two containers, then list only the ones matching a label using podman ps --filter.
podman run -d --name app-frontend --label tier=frontend my-image
podman run -d --name app-backend --label tier=backend my-image
podman ps --filter "label=tier=frontend"

### 16. Commit a modified running container into a new image with podman commit, then run a container from that image
podman commit my-container my-new-image:v1
podman run -d --name new-container my-new-image:v1

### 17. Show the filesystem changes made inside a running container with podman diff and interpret the A/C/D markers
podman diff new-container

### 18. Run a container with --network none, demonstrate it has no external connectivity, and explain a use case.
podman run -d --name my-container --network none my-image
podman exec my-container ping -c 1 8.8.8.8

### 19. Publish a port bound to a specific host IP only (-p 127.0.0.1:8082:80) and verify it is not reachable on the VM's other IPs.
podman run -d --name my-container -p 127.0.0.1:8082:8000 my-image

### 20. Use podman port to list a container's published ports and cross-check with podman inspect.
podman port my-container
podman inspect my-container

### 21. Run a container as a non-root user with --user, and verify the effective UID inside with id / whoami.
podman run --user 1001 my-image id
podman run --user 1001 my-image whoami

### 22. Generate a systemd unit (or a Quadlet .container file) for a container and explain how it makes the container start on boot.
mkdir -p ~/.config/containers/systemd/

cat << 'EOF' > ~/.config/containers/systemd/sample-app.container
Description=My Sample Podman Quadlet Application
After=network-online.target

Image=registry.access.redhat.com/ubi9/ubi-minimal:9.5
Exec=sleep infinity
ContainerName=quadlet-sample-app

WantedBy=default.target
EOF

### 23. Run podman events in one terminal while you start/stop a container in another, and describe the lifecycle events you observe.
podman events
podman run -d --name my-container my-image
podman stop my-container
podman rm my-container

## LO2
### 1. Write a Containerfile that uses an ARG to choose the base-image tag at build time (--build-arg), and build it twice with different values
nano Containerfile
ARG IMAGE_TAG=9.5-minimal
FROM registry.access.redhat.com/ubi9/ubi:${IMAGE_TAG}
RUN cat /etc/os-release | grep VERSION_ID
podman build --build-arg IMAGE_TAG=9.4 -t ubi-custom:v9.4 .

### 2. Write a multi-stage Containerfile that builds an artifact in one stage and copies only the result into a minimal final image; compare the final image size to a single-stage build.
nano Containerfile
FROM registry.access.redhat.com/ubi9/go-toolset:1.21 AS builder
RUN echo -e 'package main\nimport "fmt"\nfunc main() { fmt.Println("Hello from a minimal container!") }' > main.go
RUN go build -o myapp main.go
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
WORKDIR /app
COPY --from=builder /opt/app-root/src/myapp .
CMD ["./myapp"]
podman build -f Containerfile.multistage -t app:multi-stage .
podman images | grep app

### 3. Add a .containerignore file and demonstrate that ignored files are excluded from the build context / not copied
mkdir ignore-file && cd ignore-file
echo "console.log('App running ');" > app.js
echo "Standard public asset" > readme.md
cat << 'EOF' > .containerignore
.env
*.md
EOF
cat << 'EOF' > Containerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
WORKDIR /app
COPY . .
RUN ls -la /app
EOF
podman build -t ignore-test .

### 4. Build an image and apply three tags at once (name:1.0, name:1, name:latest); explain the registry/namespace/name:tag convention.
podman build \
  -t my-app:1.0 \
  -t my-app:1 \
  -t my-app:latest .
registry.access.redhat.com / ubi9 / ubi-minimal : 9.5
registry (registry.access.redhat.com) domain name of remote server
namespace (ubi9) user account on registry server
name (ubi-minimal) actual name of application image
tag (9.5) label appled to a specific build

### 5. Inspect an image's layer history with podman history and explain how to reduce the number of layers.
podman history registry.access.redhat.com/ubi9/ubi-minimal:9.5
podman build --squash -t my-app:v1 . 

### 6. Write a Containerfile that sets a non-root USER and a writable WORKDIR; run it and confirm whoami is not root
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
RUN microdnf update -y && \
    microdnf clean all
RUN echo "appuser:x:10001:10001::/home/appuser:/bin/bash" >> /etc/passwd && \
    echo "appuser:x:10001:" >> /etc/group && \
    mkdir -p /home/appuser /app && \
    chown -R appuser:appuser /home/appuser /app
WORKDIR /app
USER 10001
podman build -t my-app:1.0 .
podman run --rm -it my-app:1.0 whoami

### 7. Add a HEALTHCHECK to a Containerfile, run the container, and show the health status in podman ps / via podman healthcheck run.
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
RUN microdnf install -y curl && microdnf clean all
WORKDIR /var/www
HEALTHCHECK --interval=5s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8000/ || exit 1
CMD ["python3", "-m", "http.server", "8000"]
podman build -t my-image:1.0 .
podman run -d --name my-container my-imager:1.0
podman healthcheck run health-app

### 8. Demonstrate the difference between ENTRYPOINT and CMD in one Containerfile that uses both, then override CMD at podman run time
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
ENTRYPOINT ["echo", "Hello"]
CMD ["World"]
podman build -t my-image .
podman run --rm entrypoint-demo
podman run --rm entrypoint-demo "Moreno"

### 9. Explain the difference between ADD and COPY and write a Containerfile that justifies using each.
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
WORKDIR /app
COPY app-config.json . (justifies copy)
COPY server.js .
ADD dependencies/my-tools.tar.gz /usr/local/bin/  (justifies add)
CMD ["node", "server.js"]

### 10. Save an image to a tarball with podman save, remove it, then restore it with podman load; explain when you'd do this (air-gapped transfer).
podman pull registry.access.redhat.com/ubi9/ubi-minimal:9.5
podman save -o ubi9-minimal.tar registry.access.redhat.com/ubi9/ubi-minimal:9.5
podman load -i ubi9-minimal.tar
podman images | grep ubi-minimal

### 11. Pull an image by digest (name@sha256:...) and explain why it's more reproducible than a tag.
podman pull registry.access.redhat.com/ubi9/ubi-minimal@sha256:...
digests are more reprooducible bc tag is mutable

### 12. Build an image with --no-cache and explain layer caching and when disabling it helps or hurts.
podman build --no-cache -t my-image:latest
caching skips unchanged steps for speed, disabling it forces updates but can slow things down

### 13. Build from a non-default Containerfile name using -f, and explain the build-context argument.
echo "FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5" > MyCustomApp.build
podman build -f MyCustomApp.build -t my-image:1.0 .
build context (dot at the end) specifies the directory of host machine

### 14. Use COPY --chown to set file ownership in the image and verify with ls -l inside the container.
mkdir chown-demo && cd chown-demo
echo "Example data" > app.config
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
RUN echo "deploy:x:5000:5000::/home/deploy:/bin/bash" >> /etc/passwd && \
    echo "deploy:x:5000:" >> /etc/group
COPY --chown=5000:5000 app.config /app/app.config
USER 5000
podman build -t chown-test .
podman run --rm chown-test ls -l /app/app.config

### 15. Write a Containerfile combining ARG and ENV (build-time vs run-time variables) and demonstrate the difference
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
ARG BUILD_VERSION=1.0
RUN echo "Building image version: ${BUILD_VERSION}"
ENV APP_VERSION=${BUILD_VERSION}
ENV ENVIRONMENT="production"
podman build -t variable-demo:default .
podman run --rm variable-demo:default

### 16. podman login to a registry and push an image to a private repository; explain where/how the credentials are stored
podman login quay.io
podman tag security-demo:1.0 quay.io/moreno/security-demo:1.0
podman push quay.io/moreno/security-demo:1.0
credentials are stored as base64 strings inside runtime path like /run/user/1000/containers/auth.json

### 17. Remove dangling images with podman image prune and a specific image with podman rmi; explain what "dangling" means.
podman image prune

### 18. Write a Containerfile for a static file server using python -m http.server over a directory you COPY in, expose the port, and run it
mkdir public
echo "<p>Hello world</p>" > public/index.html
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
RUN microdnf install -y python3 && microdnf clean all
WORKDIR /app
COPY public/ ./public
EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080", "--directory", "public"]
podman build -t static-server:1.0 .
podman run -d --name my-web-server -p 8080:8080 static-server:1.0

### 19. Build an image that installs a specific pinned version of a package (apt or apk) and explain why pinning matters for reproducibility.
FROM alpine:3.20
RUN apk add --no-cache curl=8.11.1-r0
CMD ["curl", "--version"]
podman build -t pinned-curl .
podman run --rm pinned-curl

### 20. Inspect an unknown image with podman image inspect to discover its default Entrypoint/Cmd, exposed ports, and env, then run it accordingly.
podman image inspect registry.access.redhat.com/ubi9/ubi-minimal:9.5

### 21. Write a Containerfile with a multi-line RUN ... && ... that installs packages and cleans the package cache in the same layer; explain why cleanup must share the layer.
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
RUN microdnf install -y tar gzip && \
    microdnf clean all
CMD ["tar", "--version"]
cleanup must share the layer because layers are immutable, so cleaning up later only hides cached files instead of deleting them

### 22. Tag and push the same image to two different registries (e.g. Docker Hub and Quay) and explain when image mirroring is useful.
podman build -t my-app:1.0 .
podman tag my-app:1.0 quay.io/moreno/my-app:1.0
podman push quay.io/moreno/my-app:1.0
podman tag my-app:1.0 docker.io/moreno/my-app:1.0
podman push docker.io/moreno/my-app:1.0
Image mirroring is useful to avoid registry rate limits

### 23. Run an image and list the OS packages installed inside it; explain the security/size value of choosing a minimal base image
podman run -d --rm my-image -l
Minimal base images are more secure because they strip out explotiable binaries

## LO3
### 1. Create a pod with podman pod create (publishing a port), add an app container and a database container to it, and show they reach each other over localhost (shared network namespace).
podman pod create --name my-pod -p 8080:8080
podman run -d --name my-db \
  --pod multi-tier-pod \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=app_db \
  registry.access.redhat.com/ubi9/postgresql-15:latest
podman run -d --name my-app \
  --pod multi-tier-pod \
  registry.access.redhat.com/ubi9/ubi-minimal:9.5 \
  /bin/sh -c "while true; do sleep 30; done"
podman exec -it my-app microdnf install -y nc
podman exec -it my-app nc -zv localhost 5432

### 2. Create a user-defined network, run an app container and a postgres container on it, and prove the app resolves the database by container name (podman DNS on custom networks).
podman network create my-net
podman run -d --name my-postgres \
  --network app-net \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=test_db \
  registry.access.redhat.com/ubi9/postgresql-15:latest

podman run -d --name my-app \
  --network app-net \
  registry.access.redhat.com/ubi9/ubi-minimal:9.5 \
  /bin/sh -c "while true; do sleep 30; done"
podman exec -it my-app getent hosts my-postgres

### 3. Deploy a two-tier stack of your choice that is not Drupal/Joomla/WordPress (e.g. Ghost + MySQL, Gitea + PostgreSQL, or Redmine + PostgreSQL) and complete its first-run setup
podman network create gitea-net
podman run -d --name gitea-db \
  --network gitea-net \
  -e POSTGRES_USER=gitea \
  -e POSTGRES_PASSWORD=secret_db_pass \
  -e POSTGRES_DB=gitea \
  -v gitea-db-data:/var/lib/postgresql/data:Z \
  docker.io/library/postgres:16-alpine
podman run -d --name gitea-app \
  --network gitea-net \
  -p 3000:3000 \
  -p 2222:22 \
  -v gitea-app-data:/data:Z \
  docker.io/gitea/gitea:1.22-alpine

### 4. Persist the database's data in a named volume so it survives podman rm + re-creation of the DB container; prove the data is still there afterward.
podman volume create pg-data
podman run -d --name test-db \
  -v pg-data:/var/lib/postgresql/data:Z \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  docker.io/library/postgres:16-alpine
podman exec -it test-db psql -U admin -d mydb -c "CREATE TABLE users (id SERIAL, name TEXT); INSERT INTO users (name) VALUES ('Moreno');"
podman rm -f test-db
podman run -d --name test-db \
  -v pg-data:/var/lib/postgresql/data:Z \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  docker.io/library/postgres:16-alpine
podman exec -it test-db psql -U admin -d mydb -c "SELECT * FROM users;"

### 5. Use a podman secret (podman secret create + --secret) to pass the database password instead of --env; explain the security benefit.
echo "password123" | podman secret create db_pass -
podman run -d --name secure-db \
  --secret db_pass,type=env,target=POSTGRES_PASSWORD \
  -e POSTGRES_USER=admin \
  -e POSTGRES_DB=secure_prod \
  docker.io/library/postgres:16-alpine

### 6. Write a podman compose file (compose.yaml) for an app + database, bring it up with podman compose up -d, and show both services running
services:
- db:
    - image: docker.io/library/postgres:16-alpine
    - container_name: compose-db
    - environment:
      - POSTGRES_USER: app_user
      - POSTGRES_PASSWORD: secure_password
      - POSTGRES_DB: app_production
    - volumes:
      - db_data:/var/lib/postgresql/data:Z
    networks:
      - app-tier

- web:
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - container_name: compose-web
    - command: /bin/sh -c "while true; do sleep 30; done"
    - ports:
      - "8080:8080"
    - networks:
      - app-tier
    - depends_on:
      - db

volumes:
  - db_data:

networks:
  - app-tier:

podman compose up -d
podman compose ps

### 7. In that compose file, keep the database on an internal network with no published port and expose only the app; prove the DB isn't reachable from the host.
services:
- db:
    - image: docker.io/library/postgres:16-alpine
    - container_name: compose-db
    - environment:
      - POSTGRES_USER: app_user
      - POSTGRES_PASSWORD: secure_password
      - POSTGRES_DB: app_production
    - volumes:
      - db_data:/var/lib/postgresql/data:Z
    networks:
      - backend-tier

- web:
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - container_name: compose-web
    - command: /bin/sh -c "while true; do sleep 30; done"
    - ports:
      - "8080:8080"
    - networks:
      - backend-tier
      - frontend-tier
    - depends_on:
      - db

volumes:
  - db_data:

networks:
  - frontend-tier:
  - backend-tier:
    - internal: true

podman compose up -d

### 8. Add a compose healthcheck plus depends_on: condition: service_healthy so the app waits for the database; demonstrate the startup ordering.
services:
- db:
    - image: docker.io/library/postgres:16-alpine
    - container_name: compose-db
    - environment:
      - POSTGRES_USER: app_user
      - POSTGRES_PASSWORD: secure_password
      - POSTGRES_DB: app_production
    - volumes:
      - db_data:/var/lib/postgresql/data:Z
    networks:
      - app-tier
    healthcheck:
      - test: ["CMD-SHELL", "pg_isready -U app_user -d app_production"]
      - interval: 3s
      - timeout: 3s
      - retries: 5
      - start_period: 2s

- web:
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - container_name: compose-web
    - command: /bin/sh -c "while true; do sleep 30; done"
    - ports:
      - "8080:8080"
    - networks:
      - backend-tier
      - frontend-tier
    - depends_on:
      - db:
        - condition: service_healthy

volumes:
  - db_data:

networks:
  - app-tier

podman compose up -d

### 9. Use an env_file: in compose for the database credentials instead of inline values.
cat << EOF > .env.db
POSTGRES_USER=app_user
POSTGRES_PASSWORD=super_secure_password_123
POSTGRES_DB=app_production
EOF
services:
- db:
    - image: docker.io/library/postgres:16-alpine
    - container_name: compose-db
    - envi_file:
      - .env.db
    - volumes:
      - db_data:/var/lib/postgresql/data:Z
    networks:
      - app-tier

- web:
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - container_name: compose-web
    - command: /bin/sh -c "while true; do sleep 30; done"
    - ports:
      - "8080:8080"
    - networks:
      - app-tier
    - depends_on:
      - db

volumes:
  - db_data:

networks:
  - app-tier:

podman compose up -d

### 10. Scale one compose service to multiple replicas (podman compose up --scale app=3) and explain how requests are distributed
podman compose up -d --scale web=3
podman compose ps
Requests are distributed thru podman's internal dns, which mapps service name to ip addresses

### 11. Mount a bind mount for app configuration and a named volume for data in the same container, and explain why you'd use each.
mkdir -p ./config
echo "max_connections = 50" > ./config/custom.conf
podman volume create db-storage
podman run -d --name mixed-storage-db \
  -v ./config/custom.conf:/etc/postgresql/postgresql.conf:Z \
  -v db-storage:/var/lib/postgresql/data:Z \
  -e POSTGRES_PASSWORD=secret \
  docker.io/library/postgres:16-alpine

### 12. Back up a named volume to a tarball using a helper alpine/busybox container that mounts the volume, then restore it into a fresh volume.
podman run --rm \
  -v db-storage:/volume-data:ro \
  -v ./backup:/backup-dir:Z \
  docker.io/library/busybox \
  tar -czf /backup-dir/db_backup.tar.gz -C /volume-data .
podman volume create db-storage-restored
podman run --rm \
  -v db-storage-restored:/volume-data:Z \
  -v ./backup:/backup-dir:ro \
  docker.io/library/busybox \
  tar -xzf /backup-dir/db_backup.tar.gz -C /volume-data

### 13. Put an nginx reverse-proxy container in front of an app container on a shared network and route host traffic through nginx to the app.
podman network create proxy-net
podman run -d --name my-app-service \
  --network proxy-net \
  registry.access.redhat.com/ubi9/ubi-minimal:9.5 \
  /bin/sh -c "echo 'Hello from behind the proxy!' > index.html; exec python3 -m http.server 8080"
mkdir -p ./nginx-conf
cat << 'EOF' > ./nginx-conf/default.conf
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://my-app-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
podman run -d --name nginx-proxy \
  --network proxy-net \
  -p 8080:80 \
  -v ./nginx-conf/default.conf:/etc/nginx/conf.d/default.conf:Z \
  docker.io/library/nginx:alpine

### 14. Add a database-admin container (e.g. adminer or pgadmin) to the network and use it to inspect the database
podman run -d --name db-adminer \
  --network proxy-net \
  -p 8082:8080 \
  docker.io/library/adminer:latest

### 15. Generate Kubernetes YAML from a running pod with podman generate kube 
podman generate kube multi-tier-pod > multi-tier-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  - creationTimestamp: "2026-06-23T18:30:00Z"
  - labels:
    - app: multi-tier-pod
  - name: multi-tier-pod
spec:
  - containers:
    - env:
        - name: POSTGRES_USER
        - value: admin
        - name: POSTGRES_PASSWORD
        - value: secret
        - name: POSTGRES_DB
        - value: app_db
    - image: registry.access.redhat.com/ubi9/postgresql-15:latest
    - name: my-db
    - resources: {}
  - argument:
    - /bin/sh
    - -c
    - while true; do sleep 30; done
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - name: my-app
    - resources: {}
- ports:
  - containerPort: 8080
    hostPort: 8080
    protocol: TCP

### 16. Run a pod from a Kubernetes YAML with podman kube play, then tear it down with podman kube play --down.
podman kube play multi-tier-pod.yaml
podman pod ps
podman kube play multi-tier-pod.yaml --down

### 17. Set per-service CPU/memory limits in a compose file and verify them with podman stats.
services:
  - db:
    - image: docker.io/library/postgres:16-alpine
    - container_name: compose-db
    - environment:
      - POSTGRES_USER: app_user
      - POSTGRES_PASSWORD: secure_password
      - POSTGRES_DB: app_production
    - volumes:
      - db_data:/var/lib/postgresql/data:Z
    - networks:
      - app-tier
    - deploy:
      - resources:
        - limits:
          - cpus: '0.50'
          - memory: 256M

  - web:
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - container_name: compose-web
    - command: /bin/sh -c "while true; do sleep 30; done"
    - ports:
      - "8080:8080"
    - networks:
      - app-tier
    - deploy:
      - resources:
        - limits:
          - cpus: '0.25'
          - memory: 128M
    - depends_on:
      - db

volumes:
  - db_data:

networks:
  - app-tier:

podman compose up -d
podman stats 

### 18. Inspect a pod with podman pod inspect and show which Linux namespaces the containers share (network, IPC) and which they don't (PID by default).
podman pod inspect multi-tier-pod

### 19. Demonstrate name-based discovery failing when two containers are on different networks, then fix it by attaching them to a common network
podman network create net-backend
podman network create net-frontend
podman run -d --name isolated-db \
  --network net-backend \
  -e POSTGRES_PASSWORD=secret \
  docker.io/library/postgres:16-alpine
podman run -d --name isolated-app \
  --network net-frontend \
  registry.access.redhat.com/ubi9/ubi-minimal:9.5 \
  /bin/sh -c "while true; do sleep 30; done"
podman exec -it isolated-app ping isolated-db
podman network connect net-backend isolated-app

### 20. Attach a single container to two networks (a frontend and a backend) and explain the segmentation benefit.
podman network create net-public
podman network create net-private
podman run -d --name secure-database \
  --network net-private \
  -e POSTGRES_PASSWORD=supersecret \
  docker.io/library/postgres:16-alpine
podman run -d --name api-gateway \
  --network net-public \
  -p 8080:8080 \
  registry.access.redhat.com/ubi9/ubi-minimal:9.5 \
  /bin/sh -c "while true; do sleep 30; done"
podman network connect net-private api-gateway

### 21. Use podman compose logs and podman compose ps to observe and troubleshoot a multi-service stack.
services:
  - db:
    - image: docker.io/library/postgres:16-alpine
    - container_name: compose-db
    - environment:
      - POSTGRES_USER: app_user
      - POSTGRES_PASSWORD: secure_password
      - POSTGRES_DB: app_production
    - networks:
      - app-tier

  - web:
    - image: docker.io/library/postgres:16-alpine
    - container_name: compose-web
    - command: ["pg_isready", "-h", "wrong_database_host"]
    - networks:
      - app-tier
    - depends_on:
      - db

networks:
  - app-tier:
podman compose up -d
podman compose ps
podman compose logs web

### 22. Share configuration across two replicas of the same app via a shared named volume, and explain a risk of shared read-write storage.
services:
  - web-a:
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - container_name: compose-web-a
    - command: /bin/sh -c "echo 'Shared Setting' > /app/config/settings.txt; while true; do sleep 30; done"
    - volumes:
      - shared-config:/app/config:Z

  - web-b:
    - image: registry.access.redhat.com/ubi9/ubi-minimal:9.5
    - container_name: compose-web-b
    - command: /bin/sh -c "sleep 5; cat /app/config/settings.txt; while true; do sleep 30; done"
    - volumes:
      - shared-config:/app/config:Z
    - depends_on:
      - web-a

volumes:
  - shared-config:

podman compose up -d
podman compose logs web-b

### 23. Tear down a stack with podman compose down, then explain what happens to named volumes and how --volumes changes that.
podman compose down
podman compose down --volumes
podman compose down preserves named volumes so application data survives the teardown. Appending the --volumes flag overrides this safety feature, permanently deleting the volumes and all the stored data from the host.
