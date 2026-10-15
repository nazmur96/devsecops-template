# Phase 2: Containerise

**Goal:** every service has a small, non-root, reproducible image built from a
hand-written multi-stage Dockerfile, and the whole system runs locally with
Docker Compose, passing the CUJ.

**Why:** the image is what you'll scan, sign, promote and run everywhere.
Every weakness here (root user, build tools in the final image, floating
tags, secrets in a layer) is copied into every environment.

**Worked example rule (reference 07, ADR-0002):** delete any upstream
Dockerfiles and deployment files first. Writing them yourself is the lesson.

> **Read first:** [How code goes from your laptop to the user](https://github.com/nazmur96/devops-learning-journal/blob/main/foundations/how-code-goes-from-laptop-to-user.md).
> The **final output** decides the image: a Go binary → distroless, a `.jar`
> → a JRE image, a frontend `dist/` → served by Nginx with no Node in the
> final image. That's why every Dockerfile below is multi-stage.
>
> **Dependencies come through your artifact repository** (P4-8): set the npm
> registry / Maven mirror / `GOPROXY` / pip index in the build stage via build
> args, and pull base images through the ECR pull-through cache.

---

## The image standard (applies to every ticket below)

| Requirement | Value | Check |
|---|---|---|
| Build | Multi-stage: full toolchain to build, minimal runtime to run | `docker history` shows no compiler in the final stage |
| User | Non-root, **numeric** UID (`USER 10001`) | `docker inspect -f '{{.Config.User}}'` not empty / not `root` |
| Base images | Pinned **by digest** (`image:tag@sha256:…`) | hadolint DL3006/DL3007; grep for `@sha256` |
| Packages | Versions pinned (`apk add tini=0.19.0-r3`) | hadolint DL3018 (reference 01's pipeline caught this) |
| Process | Exec-form `ENTRYPOINT`/`CMD`, app is PID 1 or under `tini`; **not `npm start`** | `time docker stop` < 2 s |
| Port | ≥ 1024 inside the container where the app allows it | See the port-80 trap below |
| Secrets | Never in `ENV`, `ARG` or a `COPY`'d file; BuildKit `--mount=type=secret` if a build needs one | `docker history --no-trunc`, `dive` |
| Labels | `org.opencontainers.image.source`, `.revision`, `.created` | `docker inspect` |
| Size budget | Go < 30 MB · Node < 150 MB · Java < 200 MB | CI gate (P2-5) |
| `.dockerignore` | Excludes `.git`, `node_modules`, tests, `.env*`, `*.tfstate` | review |

> **The port-80 trap.** Docker lets non-root processes bind ports below 1024
> by default (it sets `net.ipv4.ip_unprivileged_port_start=0`); Kubernetes
> runtimes generally don't. A non-root image that works in `docker run` can
> crash with `permission denied` on port 80 in Kubernetes. Prefer making the
> app listen on 8080 (many services take a `-port` flag); otherwise set that
> sysctl in the pod's `securityContext.sysctls` (it's a "safe" sysctl).

---

## P2-1 Go services

**Type:** Story

```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.22@sha256:<digest> AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app .

FROM gcr.io/distroless/static-debian12:nonroot@sha256:<digest>
ARG REVISION SOURCE CREATED
LABEL org.opencontainers.image.revision=$REVISION \
      org.opencontainers.image.source=$SOURCE \
      org.opencontainers.image.created=$CREATED
COPY --from=build /out/app /app
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/app", "-port=8080"]
```

- [ ] Image < 30 MB; no shell in the final image (`docker run --entrypoint sh` fails, which is the point)
- [ ] Responds on its health endpoint; required env vars documented

---

## P2-2 Java (Spring Boot) services

**Type:** Story

```dockerfile
# syntax=docker/dockerfile:1.7
FROM maven:3.9-eclipse-temurin-17@sha256:<digest> AS build
WORKDIR /src
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn -q -B dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -q -B package -DskipTests

FROM eclipse-temurin:17-jre-alpine@sha256:<digest>
RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app app
COPY --from=build /src/target/*.jar /app/app.jar
USER 10001
# JAVA_TOOL_OPTIONS is read by the JVM itself; JAVA_OPTS only works if a start script passes it on
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError"
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar", "--server.port=8080"]
```

- [ ] Dependency layer cached separately from source (rebuild after a code change takes seconds)
- [ ] Image < 200 MB
- [ ] **Prove it:** `docker run --memory=512m` survives the CUJ without being OOM-killed. Then remove `MaxRAMPercentage` and watch it get killed while the JVM still reports free heap. Write down why
- [ ] Startup time measured per service (input for `startupProbe` in P6)

---

## P2-3 Node services

**Type:** Story

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-alpine@sha256:<digest> AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:20-alpine@sha256:<digest> AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build --if-present

FROM node:20-alpine@sha256:<digest>
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps  --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app ./
USER node
EXPOSE 8079
# node directly, not `npm start`: npm doesn't forward SIGTERM reliably
CMD ["node", "server.js"]
```

- [ ] Production dependencies only; image < 150 MB
- [ ] Backend URLs come from env vars, not baked in
- [ ] `time docker stop` returns in about a second (SIGTERM handled)

> Reference 04 used a **single-stage** Node image (build tools shipped to prod)
> and `CMD ["npm", "start"]`. Both are fixed above.

---

## P2-4 Hardening audit

**Type:** Story

- [ ] All images: non-root confirmed with `docker inspect`
- [ ] `dive <image>`: no `.git`, caches, test fixtures or secrets in any layer
- [ ] `hadolint` clean on every Dockerfile (or each ignore has a written reason)
- [ ] `trivy image --severity HIGH,CRITICAL --ignore-unfixed <image>`: zero findings, or each one triaged
- [ ] Every base image pinned by digest; no `latest` anywhere
- [ ] **Every listener probed** with the image running: the main port, and any metrics, admin or gRPC port the app can open. Debug and admin endpoints (profilers, env dumps, crash or toggle routes) answer 404 on all of them. Watch for global registries: in Go, `net/http/pprof` registers on `http.DefaultServeMux`, so any server using the default mux serves the profiler (podinfo run)
- [ ] **Read-only run with the image's own default arguments**: `docker run --read-only --cap-drop ALL --security-opt no-new-privileges <image>`, adding only the writable mounts your docs say the pod gets. Don't re-pass flags by hand: a flag typed in the test hides a mismatch between the image and the manifests (podinfo run: the image wrote to `/tmp/data` while every doc mounted `/data`)

---

## P2-5 Size-budget gate

**Type:** Task

```bash
#!/usr/bin/env bash
# scripts/check-image-size.sh <image> <max-MB>
set -euo pipefail
size=$(docker image inspect "$1" --format '{{.Size}}')
mb=$(( size / 1024 / 1024 ))
echo "$1: ${mb} MB (budget $2 MB)"
[ "$mb" -le "$2" ] || { echo "::error::$1 exceeds budget"; exit 1; }
```

- [ ] Script exits non-zero over budget; wired into CI as **blocking** in P7

---

## P2-6 [OPT] Multi-architecture images

> **Use when:** you plan Graviton (arm64) nodes, which are typically ~20 % cheaper, or developers build on Apple Silicon. Skip if every machine and node is amd64.

**Type:** Story · *Reference 06*

- [ ] `docker buildx build --platform linux/amd64,linux/arm64` produces a multi-arch manifest
- [ ] Lets you use Graviton (`t4g`/`m7g`) nodes later, typically ~20 % cheaper

---

## P2-7 Local Compose topology with network segmentation

**Type:** Story

```yaml
# compose.yaml (excerpt)
networks:
  frontend: {}
  backend: {}
  data: { internal: true }        # no route to the outside world

services:
  front-end:
    image: sockshop/front-end:dev
    networks: [frontend, backend]
    ports: ["8079:8079"]
    depends_on:
      catalogue: { condition: service_healthy }
  catalogue:
    image: sockshop/catalogue:dev
    networks: [backend, data]
    healthcheck:
      test: ["CMD", "/app", "-healthcheck"]     # distroless has no curl; use the binary or a sidecar check
      interval: 10s
      retries: 5
    depends_on:
      catalogue-db: { condition: service_healthy }
  catalogue-db:
    image: mysql:5.7@sha256:<digest>
    networks: [data]
    volumes: [catalogue-db:/var/lib/mysql]
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root
    secrets: [mysql_root]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]

secrets:
  mysql_root: { file: ./.secrets/mysql_root }   # .secrets/ is git-ignored

volumes:
  catalogue-db: {}
```

- [ ] All services, data stores and the broker defined; healthchecks everywhere; `depends_on: condition: service_healthy`
- [ ] Data stores only on the `data` network: **prove** `front-end` can't reach `catalogue-db` (`docker compose exec front-end nc -z catalogue-db 3306` fails)
- [ ] Secrets via Compose `secrets` or a git-ignored `.env`, never literals in the file
- [ ] `docker compose down && docker compose up` keeps the data
- [ ] Startup order deterministic across 3 cold starts

---

## P2-8 Script the CUJ

**Type:** Task

- [ ] `scripts/cuj.sh` (curl) or a Playwright/k6 script runs the full journey and exits non-zero on any failure
- [ ] Used from now on as the definition of "it works", locally, on KinD (P3), on EKS (P4+), and in the load test (P10)

---

## P2-9 Measure resources (feeds P0-5)

**Type:** Task

- [ ] Run the CUJ in a loop for 10 minutes; record `docker stats` (CPU %, memory) per service
- [ ] Update `docs/sizing-and-cost.md` with real numbers; these become the first requests/limits in P6

---

## P2-10 Write the containerisation playbook

**Type:** Story · *Reference 07, SOCKPLAT-82*

Turn what you learned into a reusable document, so the next service (or the
next project) takes an hour, not a day: `docs/playbooks/containerise-a-service.md`.

- [ ] Steps in order: read the code (entry point, dependencies, build, output; see the "laptop to user" article) → list env vars and ports → find or add a health endpoint → pick the base images per language → write the multi-stage Dockerfile from the standard → `.dockerignore` → run it non-root with a read-only filesystem → `time docker stop` (SIGTERM) → hadolint, Trivy, size budget → add it to Compose → CUJ
- [ ] A short "common failures and fixes" section from your own experience (port 80 as non-root, missing CA certificates in distroless, JVM memory, `npm start`)
- [ ] Real examples from this project, including the `catalogue-db` hardcoded-credentials constraint
- [ ] **Prove it:** someone else containerises one more service using only the playbook

---

## Prove it

- [ ] `docker compose up` from a clean clone → CUJ script passes
- [ ] Every image passes hadolint, Trivy (HIGH/CRITICAL), the size budget, and the non-root check
- [ ] A fake secret placed in the build context does **not** appear in any image layer (`.dockerignore` works)

## Common mistakes

| Mistake | Seen in |
|---|---|
| Single-stage image with build tools in production | 04 |
| `npm start` as PID 1 (slow, unclean shutdown) | 04 |
| Unpinned `apk add` / base images (non-reproducible builds) | 01 (caught by hadolint) |
| Deliberately old base image never updated after the demo | 01 (`node:20-alpine` old pin) |
| Non-root image binding port 80 works in Docker, fails in Kubernetes | — (common real-world trap) |
| `JAVA_OPTS` set but never passed to the JVM | 07's spec (use `JAVA_TOOL_OPTIONS`) |
