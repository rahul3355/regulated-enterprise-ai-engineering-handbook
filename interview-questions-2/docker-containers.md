# Docker & Containers Interview Questions -- 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | Docker/Containers (Dockerfile, Multi-Stage Builds, Compose, Security, Networking, Volumes, Production Patterns) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | backend-engineering/typescript/express-nestjs.md (Dockerfile examples), cicd-devops/container-scanning.md, cicd-devops/sbom-generation.md, cicd-devops/security-scanning.md, testing-and-quality/test-environments.md |
| Citi Relevance | Citi runs everything on Kubernetes/OpenShift, but Docker is the foundational container technology. Understanding Docker deeply is essential for debugging, security, and CI/CD pipeline design. |

---

## Difficulty Legend
- 🔵 **Must-Know** -- Foundational. Every candidate should know these.
- 🟡 **Medium** -- Core competency. What you'll actually be tested on.
- 🔴 **Advanced** -- Differentiator. Shows principal-engineer depth.

---

### Q1: 🔵 What is a Dockerfile and how does Docker use it to build an image?

**Strong Answer:**

A Dockerfile is a text file containing a sequential list of instructions that Docker uses to assemble a container image. Each instruction (FROM, RUN, COPY, ENV, etc.) creates a new read-only layer in the resulting image. Docker executes these instructions top-to-bottom, caching each layer by its content hash so that subsequent builds can reuse unchanged layers, dramatically speeding up the process.

The build process works as follows: Docker sends the build context (the directory you specify plus everything in it, minus what is excluded by `.dockerignore`) to the Docker daemon. The daemon then parses the Dockerfile, executes each instruction in a temporary container, and commits the resulting filesystem as a new layer. The final image is a stack of these layers plus metadata (entrypoint, exposed ports, environment variables).

```dockerfile
# Example from a NestJS banking API
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

In a banking context, Dockerfiles are the first artifact that security teams audit. Every instruction is scrutinized because it defines what runs in production -- including base OS, installed packages, and runtime user. A poorly written Dockerfile can introduce supply chain risks (pulling compromised base images) or compliance violations (running as root).

**Key Points to Hit:**
- [ ] Dockerfile is a set of sequential instructions executed top-to-bottom
- [ ] Each instruction creates a new read-only layer in the image
- [ ] Layer caching uses content hashes to skip unchanged steps
- [ ] Build context is sent to Docker daemon (controlled by .dockerignore)
- [ ] Banking context: Dockerfiles are security-audit artifacts

**Follow-Up Questions:**
1. How does Docker's layer caching work and when does it invalidate?
2. What is the difference between `COPY` and `ADD`?
3. How do you inspect the layers of a built image?

**Source:** `backend-engineering/typescript/express-nestjs.md`

---

### Q2: 🔵 What are multi-stage builds and why are they essential?

**Strong Answer:**

Multi-stage builds allow you to use multiple `FROM` instructions in a single Dockerfile, where each `FROM` starts a new build stage. You can copy artifacts from earlier stages into later ones using `COPY --from=<stage>`. The final image only includes the last stage by default, which means build tools, compilers, test frameworks, and source code never end up in production.

This is critical for several reasons. First, it dramatically reduces image size. A Node.js application that needs TypeScript compilation might require 500 MB of build dependencies, but the production runtime only needs the compiled JavaScript and production `node_modules` -- easily under 150 MB. Second, it improves security by minimizing the attack surface: fewer packages mean fewer potential vulnerabilities. Third, it separates build-time concerns from runtime concerns cleanly.

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production (final image)
FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

In banking environments, multi-stage builds are mandatory for compliance. Regulatory frameworks require minimal images to pass vulnerability scanning. An image that includes `git`, `curl`, compilers, or test dependencies will fail security gates. Multi-stage builds ensure only the production runtime is shipped.

**Key Points to Hit:**
- [ ] Multiple FROM instructions define separate build stages
- [ ] COPY --from=<stage> transfers artifacts between stages
- [ ] Final image only includes the last stage
- [ ] Reduces image size and attack surface
- [ ] Banking context: compliance requires minimal production images

**Follow-Up Questions:**
1. How do you name stages and reference them?
2. Can you stop at a specific stage during build?
3. What happens to intermediate stage images after the build?

**Source:** `backend-engineering/typescript/express-nestjs.md`, `cicd-devops/container-scanning.md`

---

### Q3: 🔵 Explain the purpose of `.dockerignore` and give examples of what should be in it.

**Strong Answer:**

The `.dockerignore` file tells Docker which files and directories to exclude from the build context before sending it to the Docker daemon. It works like `.gitignore` but applies specifically to Docker builds. Without it, Docker sends everything in the build directory, including `.git/`, `node_modules/`, local environment files, IDE configs, and test data -- bloating the build context and slowing down builds.

This file serves three purposes. First, it reduces build time by shrinking the context size sent over the Docker API. Second, it prevents sensitive data (`.env` files with database credentials, API keys, TLS certificates) from accidentally being baked into images. Third, it improves layer caching -- by excluding `node_modules/`, you ensure that `npm ci` runs fresh based on `package.json` changes rather than stale local installs.

```
# .dockerignore for a Node.js banking API
node_modules/
npm-debug.log*
.git/
.gitignore
.env
.env.*
!.env.example
docker-compose*.yaml
Dockerfile
*.md
.editorconfig
.vscode/
.idea/
coverage/
.nyc_output/
dist/
test_data/
```

In a banking context, accidentally committing secrets into images is a critical compliance violation. The `.dockerignore` is the first line of defense against this. Regulatory auditors check for its presence and content as part of the container security review.

**Key Points to Hit:**
- [ ] Excludes files from build context sent to Docker daemon
- [ ] Reduces build time and image size
- [ ] Prevents secrets and sensitive data from being baked into images
- [ ] Improves layer cache effectiveness
- [ ] Banking context: auditors verify .dockerignore as a security control

**Follow-Up Questions:**
1. What is the difference between `.dockerignore` and `.gitignore`?
2. How do you negate a pattern in `.dockerignore`?
3. Can you specify a different .dockerignore for a build?

**Source:** `cicd-devops/security-scanning.md`, `cicd-devops/container-scanning.md`

---

### Q4: 🔵 Why should you run containers as non-root users?

**Strong Answer:**

By default, processes inside a Docker container run as the root user (UID 0). If an attacker exploits a vulnerability in your application to escape the container or exploit a kernel bug, they gain root-level access on the host. Running as a non-root user limits the blast radius: even if the container is compromised, the attacker only has the privileges of that unprivileged user.

You configure non-root execution in the Dockerfile using the `USER` instruction:

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

Or at runtime: `docker run --user 1000:1000 my-image`.

In Kubernetes/OpenShift (which Citi uses), this becomes even more critical. OpenShift enforces non-root execution by default through Security Context Constraints (SCCs). If your image expects to run as root, it will fail to start. You must design your Dockerfile to create a non-root user and set appropriate file permissions during the build stage.

The banking compliance angle is direct. Regulatory frameworks (FFIEC, PCI-DSS, SOX) mandate the principle of least privilege. Running containers as root violates this principle and will fail security audits. Container scanning tools like Trivy flag root execution as a finding, and OPA policies can block deployment entirely:

```rego
# OPA policy from container-scanning.md
deny[msg] {
  input.image.user == "root"
  msg := "Container runs as root user"
}
```

**Key Points to Hit:**
- [ ] Default Docker containers run as root (UID 0)
- [ ] Non-root limits blast radius of container escape
- [ ] Use USER instruction in Dockerfile to set non-root user
- [ ] OpenShift enforces non-root via SCCs
- [ ] Banking context: least privilege is a regulatory requirement

**Follow-Up Questions:**
1. How do you handle files that need to be written by the container?
2. What user does OpenShift assign by default?
3. How do you debug a container that fails due to permission issues?

**Source:** `cicd-devops/container-scanning.md`, `backend-engineering/typescript/express-nestjs.md`

---

### Q5: 🔵 What is Docker Compose and when do you use it?

**Strong Answer:**

Docker Compose is a tool for defining and running multi-container applications. You describe your entire application stack -- web services, databases, caches, message queues -- in a single `docker-compose.yaml` file, specifying how each service is built, what ports it exposes, what environment variables it needs, and how services depend on each other. Then a single `docker compose up` command starts everything.

Compose is primarily used for local development environments. In a banking GenAI platform, you might have a RAG API, a vector database (Qdrant), PostgreSQL, Redis, a mock LLM provider, and an OpenTelemetry collector. Running all of these manually with individual `docker run` commands is error-prone and not reproducible across team members. Compose makes the entire environment declarative and version-controlled.

```yaml
# docker-compose.local.yaml (from test-environments.md)
version: '3.8'
services:
  banking-rag-api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - LLM_PROVIDER=mock
      - EMBEDDING_MODEL=all-MiniLM-L6-v2
      - VECTOR_DB_URL=http://qdrant:6333
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/banking_test
    depends_on:
      - qdrant
      - postgres
      - redis
    volumes:
      - ./app:/app

  qdrant:
    image: qdrant/qdrant:v1.7.0
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=banking_test
    volumes:
      - ./test_data/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine

volumes:
  qdrant_data:
```

In production, Citi uses Kubernetes, not Compose. But Compose is the standard for local dev, CI ephemeral environments, and integration test setups. Understanding Compose networking, volume management, and dependency ordering is essential.

**Key Points to Hit:**
- [ ] Compose defines multi-container apps in a single YAML file
- [ ] Primary use: local development and CI test environments
- [ ] Handles service dependencies, networking, volumes, and environment variables
- [ ] Production uses Kubernetes, not Compose
- [ ] Banking context: reproducible dev environments for compliance testing

**Follow-Up Questions:**
1. How does `depends_on` work and does it wait for service readiness?
2. How does Docker Compose handle networking between services?
3. Can you use Compose files in production CI/CD?

**Source:** `testing-and-quality/test-environments.md`

---

### Q6: 🟡 Compare Alpine-based images vs. slim/debian-based images. When would you choose each?

**Strong Answer:**

Alpine Linux is a minimal distribution (~5 MB) using musl libc instead of glibc. Debian slim images are stripped-down versions of full Debian (~70-100 MB) and use standard glibc. The choice has significant implications.

Alpine advantages: extremely small base image, minimal attack surface, fewer packages mean fewer vulnerabilities. Disadvantages: musl libc incompatibilities can cause subtle bugs (especially with C extensions, native binaries, or glibc-compiled dependencies like some Python wheels or Node.js native addons). In a banking API using pure TypeScript/Node.js, Alpine works perfectly. But for a Python ML service using NumPy, SciPy, or TensorFlow, you will face compilation issues or need to install build tools, defeating the size benefit.

Debian slim advantages: full glibc compatibility, wide package availability, better debugging tools available if needed. Disadvantages: larger image size, more packages mean more potential vulnerabilities to scan and manage.

For a Citi GenAI service, the decision matrix is:
- **Alpine**: Pure Node.js/Go services, CLI tools, sidecar containers where size matters most
- **Slim**: Python ML services with native dependencies, services requiring specific system libraries, services where debugging tools may be needed
- **Distroless**: Maximum security posture, no shell, minimal filesystem (covered in Q7)

```dockerfile
# Alpine -- good for Node.js
FROM node:20-alpine AS production

# Debian slim -- good for Python with native deps
FROM python:3.12-slim AS production
```

In regulated banking environments, security teams often mandate Alpine or distroless unless there is a documented technical requirement for a larger base. Every extra package is an additional vulnerability scan finding to manage.

**Key Points to Hit:**
- [ ] Alpine uses musl libc (5 MB), Debian slim uses glibc (70-100 MB)
- [ ] Alpine can have compatibility issues with native binaries/C extensions
- [ ] Slim has broader compatibility but more vulnerability surface
- [ ] Banking context: security teams mandate minimal base images
- [ ] Choice depends on application type: pure Node.js/Go = Alpine, Python ML = slim

**Follow-Up Questions:**
1. How do you debug a container with no shell?
2. What is the `ldd` command and why does it matter for musl vs glibc?
3. How do you check the vulnerability count difference between base images?

**Source:** `cicd-devops/container-scanning.md`

---

### Q7: 🟡 What are distroless images and when should you use them?

**Strong Answer:**

Distroless images, maintained by Google, contain only your application and its runtime dependencies -- no package manager, no shell, no standard Linux utilities, no `/bin/sh`. They are built from the same components as Google's production container images. A distroless Node.js image is typically 20-30 MB, compared to 40+ MB for Alpine and 100+ MB for slim.

The security implications are significant. Without a shell, an attacker who gains code execution inside the container cannot run shell commands, explore the filesystem, or install additional tools. This dramatically reduces the attack surface. For banking applications handling financial data, this is the gold standard.

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20 AS production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./
# No shell, no package manager, no root access
CMD ["dist/main.js"]
```

Trade-offs: debugging is extremely difficult because there is no shell. You cannot `docker exec -it <container> sh` to inspect state. You must rely on structured logging, health check endpoints, and observability tooling. Additionally, distroless images do not include CA certificates by default in some versions, so HTTPS calls may fail without explicitly adding them.

In a banking CI/CD pipeline, distroless images pass vulnerability scans with near-zero findings, which accelerates the compliance approval process. However, you must verify that your application and all its native dependencies work correctly on the distroless base.

**Key Points to Hit:**
- [ ] Distroless = no shell, no package manager, only app + runtime deps
- [ ] Significantly reduces attack surface (no shell for attackers)
- [ ] Harder to debug -- must rely on logging and observability
- [ ] Banking context: passes security scans with near-zero findings
- [ ] Trade-off: may need to add CA certs, native deps may not work

**Follow-Up Questions:**
1. How do you debug a production issue in a distroless container?
2. How do you add CA certificates to a distroless image?
3. Can you use distroless with Python?

**Source:** `cicd-devops/container-scanning.md`

---

### Q8: 🟡 How does Docker layer caching work and how do you optimize for it?

**Strong Answer:**

Docker caches each layer of an image build. When you rebuild, Docker checks if the instruction and all preceding layers are identical to a cached version. If yes, it reuses the cache. If any layer changes, that layer and all subsequent layers are rebuilt.

The key insight is that Docker invalidates cache based on content changes. For `COPY` and `ADD`, it uses the file checksum. For `RUN`, it uses the exact command string. For `FROM`, it uses the image digest.

Optimizing for cache means ordering your Dockerfile instructions from least-changed to most-changed:

```dockerfile
# GOOD order (cache-optimized)
FROM node:20-alpine AS builder
WORKDIR /app

# Step 1: Copy dependency manifests (change infrequently)
COPY package*.json ./

# Step 2: Install dependencies (only re-runs when package*.json changes)
RUN npm ci

# Step 3: Copy source code (changes frequently)
COPY . .

# Step 4: Build (only re-runs when source changes)
RUN npm run build

# BAD order (cache-busting)
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .              # Copies everything -- invalidates on ANY file change
RUN npm ci            # Always re-runs because COPY above changed
RUN npm run build
```

The `npm ci` pattern is critical: it installs exactly what is in `package-lock.json`, is deterministic, and is faster than `npm install`. Combined with copying only `package*.json` first, it ensures dependency installation is cached unless dependencies actually change.

In CI/CD pipelines at Citi, build times directly impact developer velocity and deployment frequency. A well-optimized Dockerfile can reduce build times from 10+ minutes to under 2 minutes, which is significant when every PR triggers a build.

**Key Points to Hit:**
- [ ] Each instruction creates a cached layer invalidated by content changes
- [ ] Order instructions from least-changed to most-changed
- [ ] Copy package files before source code to cache dependency installs
- [ ] Use npm ci for deterministic, cacheable installs
- [ ] Banking context: build time optimization impacts deployment velocity

**Follow-Up Questions:**
1. How do you force a rebuild without cache?
2. What is BuildKit and how does it improve caching?
3. How does cache work with multi-stage builds?

**Source:** `backend-engineering/typescript/express-nestjs.md`

---

### Q9: 🟡 How do you handle service dependencies and readiness in Docker Compose?

**Strong Answer:**

Docker Compose provides `depends_on` to declare service dependencies, but by default it only waits for containers to start, not for them to be ready to accept connections. This is a common trap: your API container might start before PostgreSQL has finished initialization, causing connection failures.

There are three levels of dependency handling:

**Level 1: Basic depends_on (start order only)**
```yaml
services:
  api:
    depends_on:
      - postgres
      - redis
```
This only ensures postgres and redis containers start before api. It does NOT wait for them to be ready.

**Level 2: Service healthcheck-based depends_on (Compose v2.1+)**
```yaml
services:
  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    environment:
      - POSTGRES_PASSWORD=postgres

  api:
    depends_on:
      postgres:
        condition: service_healthy
```
This waits until `pg_isready` reports PostgreSQL is actually accepting connections.

**Level 3: Application-level retry logic (most robust)**
Even with healthchecks, you should implement retry logic in your application. Database connections can fail during startup, network partitions, or scaling events. A banking API should retry with exponential backoff:

```python
import time
import psycopg2

def connect_with_retry(dsn, max_retries=10, backoff=2):
    for attempt in range(max_retries):
        try:
            return psycopg2.connect(dsn)
        except psycopg2.OperationalError:
            if attempt == max_retries - 1:
                raise
            time.sleep(backoff ** attempt)
```

In banking test environments (as documented in `test-environments.md`), the Docker Compose setup includes healthchecks for all infrastructure services (Qdrant, PostgreSQL, Redis) and the API implements retry logic. This two-layer approach ensures reliable startup even when services have different initialization times.

**Key Points to Hit:**
- [ ] depends_on only controls start order, not readiness
- [ ] Use healthcheck + condition: service_healthy for true readiness
- [ ] Application-level retry with exponential backoff is essential
- [ ] Healthchecks use interval, timeout, retries configuration
- [ ] Banking context: reliable startup is critical for automated test pipelines

**Follow-Up Questions:**
1. What is the difference between service_healthy and service_started?
2. How do you write a healthcheck for a custom service?
3. What happens if a healthcheck fails repeatedly?

**Source:** `testing-and-quality/test-environments.md`

---

### Q10: 🟡 Explain Docker networking modes: bridge, host, and overlay.

**Strong Answer:**

Docker provides several networking drivers, each suited to different scenarios:

**Bridge (default):** Creates an internal virtual network (`docker0` bridge) on the host. Each container gets its own network namespace with a unique IP address on the bridge. Containers communicate through this bridge using NAT. Port mapping (`-p 8080:3000`) forwards host port 8080 to container port 3000. DNS resolution between containers on the same user-defined bridge network uses container names as hostnames.

```bash
# Create a custom bridge network
docker network create banking-net

# Run containers on the same network
docker run -d --name postgres --network banking-net postgres:16-alpine
docker run -d --name api --network banking-net -p 8080:3000 my-api

# Inside api container, resolve postgres by name
# postgres resolves to the container's IP on banking-net
```

**Host:** Removes network isolation entirely. The container shares the host's network namespace directly. No port mapping needed -- the container binds directly to host ports. Performance is marginally better (no NAT overhead), but you lose port isolation. Two containers cannot bind to the same host port. Rarely used in production due to security concerns in banking environments.

```bash
docker run -d --network host my-api  # Binds directly to host ports
```

**Overlay:** Used in Docker Swarm (and conceptually similar to Kubernetes CNI). Spans multiple Docker hosts, allowing containers on different physical machines to communicate as if on the same network. Uses VXLAN encapsulation. At Citi, Kubernetes handles this with its CNI plugins (Calico, Cilium), but the concept is the same.

```bash
docker network create -d overlay --attachable swarm-net
```

**Key networking concepts for banking:**
- **DNS resolution:** Docker's embedded DNS server resolves container names to IPs on user-defined networks
- **Network isolation:** Sensitive services (databases) should be on separate networks from public-facing APIs
- **Port mapping:** Only expose necessary ports; in production, use internal networks with no host port exposure

**Key Points to Hit:**
- [ ] Bridge: default, isolated virtual network with NAT and port mapping
- [ ] Host: shares host network namespace, no isolation, rarely used in banking
- [ ] Overlay: multi-host networking (Swarm), analogous to Kubernetes CNI
- [ ] User-defined bridges provide automatic DNS resolution by container name
- [ ] Banking context: network isolation between public and internal services

**Follow-Up Questions:**
1. How does Docker's embedded DNS server work?
2. Can containers on different networks communicate?
3. How does network isolation work in Docker Compose?

**Source:** `testing-and-quality/test-environments.md`

---

### Q11: 🟡 What are Docker volumes and how do they differ from bind mounts?

**Strong Answer:**

Docker provides three mechanisms for persisting data and sharing files with containers:

**Named Volumes:** Managed entirely by Docker, stored in Docker's storage directory (`/var/lib/docker/volumes/` on Linux). They are the preferred mechanism for persisting database data, application state, and any data that should survive container recreation.

```yaml
volumes:
  qdrant_data:  # Named volume declaration

services:
  qdrant:
    image: qdrant/qdrant:v1.7.0
    volumes:
      - qdrant_data:/qdrant/storage  # Named volume mount
```

**Bind Mounts:** Map a specific host filesystem path into the container. The path must exist on the host. Primary use case is local development, where you mount your source code so that changes are reflected in the container without rebuilding.

```yaml
services:
  banking-rag-api:
    volumes:
      - ./app:/app        # Bind mount: host ./app -> container /app
      - ./config:/config   # Bind mount: host ./config -> container /config
```

**tmpfs Mounts:** Stores data in the host's memory only. Nothing is written to the filesystem. Useful for sensitive data that should never touch disk (encryption keys, tokens) or for high-performance scratch space.

```yaml
services:
  api:
    tmpfs:
      - /tmp/secrets:size=64m
```

Key differences:
| Feature | Named Volumes | Bind Mounts |
|---------|--------------|-------------|
| Managed by | Docker | User |
| Location | /var/lib/docker/volumes/ | Any host path |
| Portability | High (Docker manages) | Low (host-path dependent) |
| Best for | Production data, databases | Local dev, source code |

In banking environments, named volumes are used in production (via Kubernetes PersistentVolumeClaims) while bind mounts are restricted to local development. tmpfs mounts are used for sensitive runtime data that must never persist to disk, aligning with data protection regulations.

**Key Points to Hit:**
- [ ] Named volumes: Docker-managed, portable, for production data
- [ ] Bind mounts: host-path mapped, for local dev source code hot-reload
- [ ] tmpfs: in-memory only, for sensitive data that must not touch disk
- [ ] Banking context: bind mounts in dev, named volumes in production
- [ ] tmpfs aligns with data protection regulations for ephemeral secrets

**Follow-Up Questions:**
1. How do you back up a named volume?
2. What happens to volumes when you run docker compose down?
3. How do volumes work in Kubernetes vs Docker?

**Source:** `testing-and-quality/test-environments.md`

---

### Q12: 🟡 How do you integrate Docker into a CI/CD pipeline?

**Strong Answer:**

Docker image building is a core CI/CD pipeline stage. At Citi, every PR and merge triggers a container build, scan, and push. The pipeline typically follows this flow:

**1. Build the image:**
```yaml
- name: Build image
  run: docker build -t genai-api:${{ github.sha }} .
```

**2. Scan for vulnerabilities (mandatory gate):**
```yaml
- name: Run Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'genai-api:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
    ignore-unfixed: true
```

**3. Generate SBOM (compliance requirement):**
```yaml
- name: Generate Image SBOM
  uses: anchore/sbom-action@v0
  with:
    image: quay.io/banking/genai-api:${{ github.sha }}
    format: cyclonedx-json
    output-file: sbom-image.json
```

**4. Push to registry with proper tagging:**
```yaml
- name: Push to registry
  run: |
    docker tag genai-api:${{ github.sha }} quay.io/banking/genai-api:${{ github.sha }}
    docker tag genai-api:${{ github.sha }} quay.io/banking/genai-api:latest
    docker push quay.io/banking/genai-api:${{ github.sha }}
    docker push quay.io/banking/genai-api:latest
```

**Tagging strategy matters:**
- **SHA/commit hash**: Immutable, traces image to exact code version (primary tag)
- **Semantic version**: Human-readable release tags (v1.2.3)
- **latest**: Convenient but dangerous in production -- never use in K8s deployments
- **branch name**: For ephemeral environment deployments

**Upload scan results to security dashboard:**
```yaml
- name: Generate SARIF report
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'genai-api:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

In banking CI/CD, the pipeline must enforce quality gates: CRITICAL vulnerabilities block merges, HIGH vulnerabilities require SLA-tracked remediation. The SBOM is stored alongside the image for compliance auditing and supply chain security traceability.

**Key Points to Hit:**
- [ ] Build -> Scan -> Generate SBOM -> Tag -> Push
- [ ] Tagging strategy: SHA (immutable), semver (releases), avoid latest in prod
- [ ] CRITICAL vulnerabilities block merges (quality gate)
- [ ] SBOM generated and stored for compliance
- [ ] Banking context: security dashboard integration, SARIF reports

**Follow-Up Questions:**
1. Why should you avoid the `latest` tag in production?
2. How do you handle Docker layer caching in CI runners?
3. What is the difference between docker build and docker buildx?

**Source:** `cicd-devops/container-scanning.md`, `cicd-devops/sbom-generation.md`

---

### Q13: 🟡 How do you configure health checks, restart policies, and resource limits for production containers?

**Strong Answer:**

Production containers require three categories of operational configuration:

**Health Checks:** Tell the orchestrator whether the container is functioning correctly. There are two types: liveness (is the process alive?) and readiness (is it ready to accept traffic?).

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

In Kubernetes, these are defined as pod spec probes:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  periodSeconds: 10
```

**Restart Policies:** Define what happens when a container exits.
- `no`: Never restart (default)
- `always`: Always restart, even on manual stop
- `unless-stopped`: Always restart unless manually stopped
- `on-failure[:max-retries]`: Restart only on non-zero exit codes

```yaml
# Docker Compose
services:
  api:
    restart: unless-stopped
```

In Kubernetes, restartPolicy is always managed by the controller (Deployment = Always, Job/CronJob = OnFailure/Never).

**Resource Limits:** Prevent a single container from consuming all host resources. Essential in multi-tenant banking environments.

```yaml
# Docker Compose
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 512M
```

In Kubernetes (Citi's production environment):
```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "4Gi"
    nvidia.com/gpu: 1
```

**Logging Drivers:** Configure how container stdout/stderr is captured and forwarded.
```yaml
# Docker Compose
services:
  api:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

In production at Citi, logs flow through Fluentd/Fluent Bit to centralized logging platforms (Splunk, ELK) for compliance retention and audit trail.

**Key Points to Hit:**
- [ ] Health checks: liveness (is alive) vs readiness (can accept traffic)
- [ ] Restart policies: unless-stopped for Compose, managed by K8s controllers
- [ ] Resource limits prevent resource starvation in multi-tenant environments
- [ ] Requests (guaranteed) vs limits (maximum) in Kubernetes
- [ ] Banking context: logging drivers integrate with centralized compliance logging

**Follow-Up Questions:**
1. What happens when a container exceeds its memory limit?
2. How do you configure different logging drivers for different environments?
3. What is the OOMKill and how do you prevent it?

**Source:** `testing-and-quality/test-environments.md`, `cicd-devops/container-scanning.md`

---

### Q14: 🟡 How do you scan container images for vulnerabilities and enforce policies?

**Strong Answer:**

Container scanning identifies known CVEs in the packages and dependencies within your container image. This is a mandatory compliance step in banking CI/CD pipelines.

**Tool: Trivy** (most widely used in banking):
```yaml
- name: Run Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'genai-api:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
    ignore-unfixed: true
```

Trivy scans the OS packages (apt/apk), application dependencies (npm/pip/maven), configuration files, and secrets. It maintains an up-to-date vulnerability database.

**Policy Enforcement with OPA:** Scanning alone is not enough -- you need automated gates that block non-compliant images from reaching production.

```rego
# policy.rego
package container_security

deny[msg] {
  input.vulnerabilities[_].severity == "CRITICAL"
  msg := sprintf("CRITICAL vulnerability found: %s in %s",
    [input.vulnerabilities[_].vulnerabilityID,
     input.vulnerabilities[_].pkgName])
}

deny[msg] {
  count([v | v = input.vulnerabilities[_]; v.severity == "HIGH"]) > 5
  msg := "More than 5 HIGH vulnerabilities found"
}

deny[msg] {
  input.image.user == "root"
  msg := "Container runs as root user"
}
```

**SARIF Reporting for Security Dashboards:**
Scan results are uploaded to centralized security dashboards (GitHub Security, GitLab Vulnerability Management, or enterprise tools like Snyk, Aqua Security) in SARIF format for tracking and audit.

**Banking-specific practices:**
- CRITICAL vulnerabilities must be fixed before merge (hard gate)
- HIGH vulnerabilities tracked with defined SLA (e.g., 7 days)
- Vulnerability exceptions require security team approval and documented risk acceptance
- Base images must be regularly updated (weekly/monthly cadence)
- Runtime scanning enabled for deployed containers

The full scanning pipeline integrates SAST, SCA, DAST, secret detection, IaC scanning, and container scanning as documented in `cicd-devops/security-scanning.md`.

**Key Points to Hit:**
- [ ] Trivy scans OS packages, app dependencies, configs, and secrets
- [ ] OPA policies enforce automated quality gates (CRITICAL blocks merge)
- [ ] SARIF format integrates with security dashboards
- [ ] Banking: exception process with risk acceptance for unavoidable vulnerabilities
- [ ] Both build-time and runtime scanning required

**Follow-Up Questions:**
1. How do you handle false positives from Trivy?
2. What is the difference between fixed and unfixed vulnerabilities?
3. How does runtime scanning differ from build-time scanning?

**Source:** `cicd-devops/container-scanning.md`, `cicd-devops/security-scanning.md`

---

### Q15: 🟡 What is an SBOM and why is it critical for banking container security?

**Strong Answer:**

A Software Bill of Materials (SBOM) is a comprehensive, machine-readable inventory of all components, libraries, and dependencies in a software artifact. For container images, it lists every package (OS-level and application-level) with its version, license, and supply chain identifier (purl).

**Why it matters for banking:**
1. **Supply chain security:** When a new CVE is announced (e.g., Log4Shell), you query your SBOM repository to instantly identify all affected images and services. Without SBOMs, this requires manual investigation of every service -- a process that takes days or weeks.
2. **Compliance:** Executive Order 14028 (US), EU Cyber Resilience Act, and FFIEC guidelines increasingly require SBOMs for software procured by government and financial institutions.
3. **License compliance:** SBOMs reveal all dependency licenses, enabling automated detection of GPL or other restrictive licenses in proprietary banking software.

**Formats:**
- **CycloneDX** (OWASP): JSON/XML, recommended for banking, supports vulnerability and license data
- **SPDX** (Linux Foundation): JSON/RDF/tag, broader ecosystem support
- **Syft** (Anchore): Tool-specific, can output CycloneDX

**Generation pipeline:**
```yaml
# Generate both application-level and image-level SBOMs
- name: Generate Python SBOM
  run: |
    pip install cyclonedx-bom
    cyclonedx-py --requirements requirements.txt \
      --output sbom-cyclonedx.json --output-format json

- name: Generate Image SBOM
  uses: anchore/sbom-action@v0
  with:
    image: quay.io/banking/genai-api:${{ github.sha }}
    format: cyclonedx-json
    output-file: sbom-image.json
```

**SBOM usage after generation:**
- Store alongside build artifacts in artifact registry
- Link to deployment records for traceability
- Cross-reference with vulnerability scanners for automated impact analysis
- Retain for compliance audit periods (7+ years in banking)

At Citi, SBOMs are part of the software supply chain security framework. Every image pushed to the internal registry must have an associated SBOM, and the registry can reject images without one.

**Key Points to Hit:**
- [ ] SBOM = complete inventory of all dependencies in an artifact
- [ ] Critical for rapid vulnerability impact analysis (Log4Shell example)
- [ ] CycloneDX preferred for banking; SPDX also used
- [ ] Regulatory requirements: EO 14028, EU CRA, FFIEC
- [ ] Banking context: mandatory for internal registry, linked to deployments

**Follow-Up Questions:**
1. How do you merge SBOMs from multiple sources (app + image)?
2. What is a purl and why is it important?
3. How do you validate SBOM completeness?

**Source:** `cicd-devops/sbom-generation.md`, `cicd-devops/container-scanning.md`

---

### Q16: 🔴 How would you design a container security strategy for an air-gapped banking environment?

**Strong Answer:**

An air-gapped environment has no direct internet connectivity, which fundamentally changes the container supply chain. All images, base images, vulnerability databases, and tools must be transferred through controlled, audited channels.

**Architecture:**

1. **Air-gapped internal registry:** A Harbor or Artifactory instance inside the secure network serves as the single source of truth for all container images. No pulls from Docker Hub or public registries are permitted.

2. **Approved base image catalog:** Security team pre-vetted and signed a catalog of approved base images (distroless, Alpine, specific Debian versions). These are imported into the air-gapped registry through a secure transfer process (e.g., encrypted USB media scanned in a DMZ).

3. **Offline vulnerability database:** Trivy, Grype, and other scanners require vulnerability databases. In air-gapped environments, these databases are periodically updated by:
   - Downloading the latest database on an internet-connected machine
   - Transferring through the secure channel with integrity verification
   - Importing into the air-gapped scanning infrastructure

4. **Image signing and verification:**
   ```bash
   # Sign image on internet-connected build machine
   cosign sign --key cosign.key quay.io/banking/genai-api:v1.0.0

   # Verify in air-gapped environment before deployment
   cosign verify --key cosign.pub quay.io/banking/genai-api:v1.0.0
   ```

5. **Supply chain attestation:** Each image includes provenance metadata (SLSA, in-toto attestations) proving it was built from verified source code through the approved CI/CD pipeline.

6. **Policy enforcement:** OPA/Conftest policies verify:
   - Image is from the approved internal registry only
   - Image signature is valid and signed by an authorized key
   - SBOM is present and matches the image
   - No CRITICAL vulnerabilities per the latest offline vulnerability database
   - Base image is from the approved catalog

**Operational considerations:**
- Vulnerability database updates happen on a defined schedule (weekly)
- Emergency CVE response process for zero-days (out-of-band database update)
- All image imports/exports are logged and audited
- Developers build against the air-gapped registry -- no fallback to public registries

This level of rigor is standard in tier-1 banking environments handling SWIFT, trading, or core banking workloads.

**Key Points to Hit:**
- [ ] Internal registry (Harbor/Artifactory) as single image source
- [ ] Approved base image catalog, imported through secure channels
- [ ] Offline vulnerability database with scheduled updates
- [ ] Image signing with cosign, verified before deployment
- [ ] SLSA/in-toto supply chain attestation

**Follow-Up Questions:**
1. How do you handle an emergency zero-day CVE in an air-gapped environment?
2. What is SLSA and how does it apply to containers?
3. How do you manage cosign key rotation?

**Source:** `cicd-devops/container-scanning.md`, `cicd-devops/sbom-generation.md`

---

### Q17: 🔴 Explain container image signing and verification with cosign. Why does it matter for banking?

**Strong Answer:**

Container image signing cryptographically proves that an image was built by a trusted party and has not been tampered with. Without signing, anyone with registry access can push a malicious image under a legitimate tag -- a classic supply chain attack vector.

**Cosign** (part of sigstore) is the industry standard for container image signing. It uses public-key cryptography to sign image digests and stores signatures as OCI artifacts alongside the image.

**Signing workflow:**
```bash
# 1. Generate a key pair (done once, keys stored securely)
cosign generate-key-pair

# 2. Build and push image
docker build -t quay.io/banking/genai-api:v1.0.0 .
docker push quay.io/banking/genai-api:v1.0.0

# 3. Sign the image
cosign sign --key cosign.key quay.io/banking/genai-api:v1.0.0
# Enter password for private key

# 4. Verify before deployment (CI/CD gate)
cosign verify --key cosign.pub quay.io/banking/genai-api:v1.0.0
```

**Keyless signing with OIDC:** Cosign supports keyless signing using short-lived certificates tied to an OIDC identity (e.g., CI pipeline identity). This eliminates the need to manage long-lived signing keys:
```bash
COSIGN_EXPERIMENTAL=1 cosign sign quay.io/banking/genai-api:v1.0.0
```

**Banking context:**
- **Regulatory requirement:** FFIEC and PCI-DSS require cryptographic integrity verification of deployed software
- **Supply chain attack prevention:** Without signing, a compromised CI/CD token or registry credential allows an attacker to inject malicious images
- **Audit trail:** Signatures provide non-repudiation -- you can prove exactly who built and signed each image
- **Admission control:** Kubernetes admission controllers (e.g., Kyverno, OPA Gatekeeper) can enforce that only signed images are deployed

```yaml
# Kyverno policy: only allow signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  rules:
    - name: verify-image-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["quay.io/banking/*"]
          attestors:
            - count: 1
              entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

**Key management:** Signing keys must be stored in HSMs (Hardware Security Modules) or cloud KMS (AWS KMS, GCP KMS, Azure Key Vault), never in plaintext. Key rotation policies must be defined, and compromised keys must be immediately revoked and re-issued.

**Key Points to Hit:**
- [ ] Cosign signs image digests using public-key cryptography
- [ ] Keyless signing with OIDC eliminates long-lived key management
- [ ] Banking: non-repudiation, supply chain attack prevention, regulatory compliance
- [ ] Kubernetes admission controllers enforce signed-image-only policies
- [ ] Keys stored in HSM/KMS, with rotation and revocation procedures

**Follow-Up Questions:**
1. What is the difference between cosign sign and cosign attest?
2. How do you rotate signing keys without breaking existing deployments?
3. What happens if a signing key is compromised?

**Source:** `cicd-devops/container-scanning.md`, `cicd-devops/security-scanning.md`

---

### Q18: 🔴 How do you optimize container image size and what are the trade-offs?

**Strong Answer:**

Image size matters for three reasons: faster deployments (less network transfer), smaller attack surface (fewer packages = fewer CVEs), and lower storage costs in registries. However, optimization involves trade-offs.

**Optimization techniques (in order of impact):**

**1. Multi-stage builds (highest impact, 60-80% reduction):**
```dockerfile
# Before: ~900 MB
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/main.js"]

# After: ~120 MB
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/main.js"]
```

**2. Distroless base images (further 60-70% reduction):**
Replace `node:20-alpine` (~45 MB) with `gcr.io/distroless/nodejs20` (~20 MB).

**3. Layer squashing (marginal, for extreme optimization):**
```dockerfile
# Combine RUN instructions to reduce layers
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

**4. .dockerignore (prevents unnecessary context):**
Exclude `node_modules/`, `.git/`, test files, documentation.

**5. Alpine-specific optimizations:**
```dockerfile
# Use apk with --no-cache to avoid package index storage
RUN apk add --no-cache curl ca-certificates
```

**Trade-offs:**
| Technique | Size Savings | Debug Difficulty | Compatibility Risk |
|-----------|-------------|-----------------|-------------------|
| Multi-stage | High (60-80%) | Low | Low |
| Distroless | Very High | Very High | Medium |
| Alpine | Medium | Low | Medium (musl) |
| Layer squashing | Low | None | Low |
| Slim Debian | Medium | Low | None |

**Analysis tools:**
```bash
# Inspect image layers
docker history quay.io/banking/genai-api:v1.0.0

# Detailed layer analysis
dive quay.io/banking/genai-api:v1.0.0

# Compare image sizes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

**Banking context:** Smaller images pass vulnerability scanning more easily, which directly impacts deployment velocity. A 900 MB image might have 200+ CVEs; a 50 MB distroless image might have zero. For compliance-heavy environments, this is not an optimization -- it is a requirement.

**Key Points to Hit:**
- [ ] Multi-stage builds give 60-80% size reduction (highest impact)
- [ ] Distroless images maximize security but make debugging very difficult
- [ ] Alpine saves size but introduces musl libc compatibility risks
- [ ] Use tools like `dive` to analyze layer contents
- [ ] Banking context: smaller images = fewer CVEs = faster compliance approval

**Follow-Up Questions:**
1. What is `dive` and how do you use it?
2. When would you NOT want to optimize for size?
3. How does BuildKit's layer handling differ from classic Docker?

**Source:** `cicd-devops/container-scanning.md`, `backend-engineering/typescript/express-nestjs.md`

---

### Q19: 🔴 How do you handle secrets in containerized applications securely?

**Strong Answer:**

Secrets (database passwords, API keys, TLS certificates, OAuth client secrets) must never be baked into images or passed as plain environment variables in compose files. Banking regulations (PCI-DSS, SOX, GDPR) mandate encryption at rest, access logging, and rotation for all credentials.

**Anti-patterns (NEVER do these):**
```dockerfile
# BAD: Secrets in Dockerfile -- baked into image layers
ENV DB_PASSWORD=supersecret123
ENV API_KEY=sk-abc123

# BAD: Secrets in docker-compose.yaml -- committed to git
services:
  api:
    environment:
      - DB_PASSWORD=supersecret123
```

**Secure approaches:**

**1. Docker Secrets (Swarm mode only):**
```yaml
services:
  api:
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    external: true
```

**2. Environment files with restricted permissions (local dev):**
```yaml
services:
  api:
    env_file:
      - path: .env.local
        required: true
```
The `.env.local` file is in `.dockerignore` and `.gitignore` with filesystem permissions set to `600`.

**3. External secret managers (production at Citi):**
- **HashiCorp Vault:** Industry standard. Containers authenticate via Kubernetes ServiceAccount tokens, retrieve secrets at runtime, and secrets are injected as environment variables or mounted files.
- **AWS Secrets Manager / GCP Secret Manager:** Cloud-native alternatives, integrated with IAM.
- **Kubernetes Secrets:** Base64-encoded (NOT encrypted by default). In production, use Sealed Secrets, External Secrets Operator, or KMS-encrypted etcd.

**4. Runtime secret injection with sidecar:**
```yaml
# Kubernetes: External Secrets Operator pattern
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: banking-api-secrets
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: banking-api-secret
  data:
    - secretKey: db-password
      remoteRef:
        key: secret/data/banking/api
        property: db-password
```

**Secret rotation:** In banking, secrets must be rotated on a schedule (typically 90 days). The application must support runtime secret refresh without restart. This is achieved by:
- Mounting secrets as files (not env vars) so the sidecar can update them
- Application watches the file for changes and reloads configuration
- Vault Agent or External Secrets Operator handles the rotation

**Audit requirements:** Every secret access is logged. Who accessed what, when, and from which pod. This is non-negotiable for banking compliance.

**Key Points to Hit:**
- [ ] Never bake secrets into images or pass as plain env vars
- [ ] Docker Secrets for Swarm, Vault/Cloud SM for production
- [ ] K8s External Secrets Operator injects secrets from Vault
- [ ] Secret rotation: mount as files, app watches for changes
- [ ] Banking context: audit logging of all secret access, mandatory rotation schedules

**Follow-Up Questions:**
1. Why are environment variables less secure than file-mounted secrets?
2. How do you handle secret rotation without restarting the container?
3. What is the difference between Sealed Secrets and External Secrets?

**Source:** `cicd-devops/security-scanning.md`, `cicd-devops/container-scanning.md`

---

### Q20: 🔴 Explain how Docker container isolation works at the Linux kernel level. What are namespaces and cgroups?

**Strong Answer:**

Docker containers are not a separate technology -- they are a packaging of standard Linux kernel features: **namespaces** for isolation and **cgroups** (control groups) for resource management. Understanding this is critical for debugging container behavior, especially in production incidents.

**Namespaces (isolation):** Each namespace provides a separate view of a system resource. When Docker creates a container, it creates new namespaces for that container:

| Namespace | Isolates | Flag |
|-----------|----------|------|
| PID | Process IDs (container sees PID 1 as its init) | `--pid` |
| NET | Network stack (interfaces, ports, routing) | `--network` |
| MNT | Filesystem mount points | `--mount` |
| UTS | Hostname and domain name | `--hostname` |
| IPC | Inter-process communication (shared memory, semaphores) | `--ipc` |
| USER | User/group ID mapping | `--userns-remap` |
| TIME | Clock (monotonic and boot time) | -- |

For example, the PID namespace means processes inside a container see themselves as PID 1, even though on the host they have a completely different PID. The NET namespace gives the container its own network interfaces and routing table.

**Cgroups (resource limits):** Control groups limit how much of a resource a container can consume:

```bash
# What Docker sets under the hood
/sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
/sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
```

| Resource | Cgroup | Docker flag |
|----------|--------|-------------|
| Memory | memory | `--memory 4g` |
| CPU | cpu, cpuacct | `--cpus 2.0` |
| I/O | blkio | `--device-read-bps` |
| PIDs | pids | `--pids-limit` |

**Key debugging implications:**
- When a container is **OOMKilled**, it is the cgroup memory limit that was exceeded, not the host memory
- **PID exhaustion** inside a container does not affect the host (separate PID namespace)
- **Network debugging**: `docker exec` enters the container's NET namespace; from the host, you can find the container's veth pair with `docker inspect`
- **Filesystem**: Each container gets its own MNT namespace with overlayfs stacking image layers on top of a writable layer

**Container escape risks:** A container escape occurs when a process breaks out of its namespace isolation to access the host. This happens through:
- Kernel vulnerabilities (e.g., runc CVE-2019-5736)
- Privileged containers (`--privileged` disables all namespace isolation)
- Mounting host filesystem paths (e.g., `-v /:/host`)
- Capabilities abuse (e.g., `CAP_SYS_ADMIN` allows mount operations)

**Banking security posture:** Privileged containers are universally banned in banking environments. Capability lists are explicitly defined (drop ALL, add only what is needed). AppArmor/SELinux profiles provide mandatory access control on top of namespace isolation.

```yaml
# Secure container configuration
securityContext:
  privileged: false
  runAsNonRoot: true
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE  # Only if binding to ports < 1024
  readOnlyRootFilesystem: true
  seccompProfile:
    type: RuntimeDefault
```

**Key Points to Hit:**
- [ ] Namespaces provide isolation (PID, NET, MNT, UTS, IPC, USER, TIME)
- [ ] Cgroups provide resource limits (memory, CPU, I/O, PIDs)
- [ ] OOMKill is a cgroup event, not a host event
- [ ] Container escape via kernel bugs, privileged mode, or capability abuse
- [ ] Banking context: privileged containers banned, capabilities dropped by default

**Follow-Up Questions:**
1. What is overlayfs and how does Docker use it for the container filesystem?
2. How does `--userns-remap` enhance security?
3. What is seccomp and how does it complement namespace isolation?

**Source:** `cicd-devops/container-scanning.md`, `cicd-devops/security-scanning.md`

---

## Summary Table

| # | Question | Topic | Difficulty |
|---|----------|-------|------------|
| Q1 | What is a Dockerfile and how does Docker build images? | Dockerfile basics | 🔵 Must-Know |
| Q2 | What are multi-stage builds and why are they essential? | Multi-stage builds | 🔵 Must-Know |
| Q3 | What is .dockerignore and what should be in it? | Dockerfile best practices | 🔵 Must-Know |
| Q4 | Why run containers as non-root? | Container security | 🔵 Must-Know |
| Q5 | What is Docker Compose and when do you use it? | Docker Compose | 🔵 Must-Know |
| Q6 | Alpine vs slim base images: comparison | Image optimization | 🟡 Medium |
| Q7 | What are distroless images and when to use them? | Image optimization | 🟡 Medium |
| Q8 | How does Docker layer caching work? | Dockerfile optimization | 🟡 Medium |
| Q9 | Service dependencies and readiness in Compose | Docker Compose | 🟡 Medium |
| Q10 | Docker networking: bridge, host, overlay | Container networking | 🟡 Medium |
| Q11 | Volumes vs bind mounts vs tmpfs | Container volumes | 🟡 Medium |
| Q12 | Docker in CI/CD pipelines | CI/CD integration | 🟡 Medium |
| Q13 | Health checks, restart policies, resource limits | Production patterns | 🟡 Medium |
| Q14 | Container vulnerability scanning and policy enforcement | Container security | 🟡 Medium |
| Q15 | What is SBOM and why is it critical for banking? | CI/CD integration | 🟡 Medium |
| Q16 | Container security in air-gapped environments | Banking context | 🔴 Advanced |
| Q17 | Image signing and verification with cosign | Supply chain security | 🔴 Advanced |
| Q18 | Image size optimization techniques and trade-offs | Image optimization | 🔴 Advanced |
| Q19 | Secure secrets handling in containers | Container security | 🔴 Advanced |
| Q20 | How container isolation works (namespaces, cgroups) | Container internals | 🔴 Advanced |
