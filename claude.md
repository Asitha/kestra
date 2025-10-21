# Kestra Docker Separation Project - Context Document

**Last Updated:** 2025-10-21
**Branch:** `claude/setup-kestra-docker-011CUKtn4M8CaoW8SBpo8uPh`
**Goal:** Create separate multistage Dockerfiles for each Kestra component (executor, worker, scheduler, webserver)

---

## Project Overview

**Kestra** is a workflow orchestration platform built with Java (Gradle) and Micronaut framework. Currently uses a single Docker image with different commands to run different components. This project creates separate Dockerfiles for distributed deployments.

---

## Kestra Architecture Components

Kestra has 4 main server components that can run separately in distributed mode:

### 1. **Executor** (✅ COMPLETED)
- **Command:** `server executor`
- **Java Class:** `io.kestra.cli.commands.servers.ExecutorCommand`
- **Location:** `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/ExecutorCommand.java`
- **Purpose:** Processes and executes workflow tasks
- **CLI Options:**
  - `--skip-executions` - Skip specific execution IDs
  - `--skip-flows` - Skip specific flows (tenant|namespace|flowId)
  - `--skip-namespaces` - Skip namespaces (tenant|namespace)
  - `--skip-tenants` - Skip tenants
  - `--start-executors` - Start specific Kafka Stream executors (Kafka queue only)
  - `--not-start-executors` - Don't start specific executors (Kafka queue only)

### 2. **Worker** (⏳ TODO)
- **Command:** `server worker`
- **Java Class:** `io.kestra.cli.commands.servers.WorkerCommand`
- **Location:** `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/WorkerCommand.java`
- **Purpose:** Processes worker tasks
- **CLI Options:**
  - `-t, --thread` - Max number of worker threads (default: 8 × CPU cores)
  - `-g, --worker-group` - Worker group key (must match `[a-zA-Z0-9_-]+`) (EE only)

### 3. **Scheduler** (⏳ TODO)
- **Command:** `server scheduler`
- **Java Class:** `io.kestra.cli.commands.servers.SchedulerCommand`
- **Location:** `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/SchedulerCommand.java`
- **Purpose:** Schedules workflows based on triggers
- **CLI Options:** None (uses base server options)

### 4. **Webserver** (⏳ TODO)
- **Command:** `server webserver`
- **Java Class:** `io.kestra.cli.commands.servers.WebServerCommand`
- **Location:** `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/WebServerCommand.java`
- **Purpose:** Provides UI and API endpoints
- **CLI Options:**
  - `--no-tutorials` - Disable auto-loading tutorial flows
  - `--no-indexer` - Disable embedded indexer
  - `--skip-indexer-records` - Skip specific indexer records (comma-separated)

### Other Commands
- **Standalone:** `server standalone` - Runs all components together (current docker-compose default)
- **Local:** `server local` - Development server with no dependencies
- **Indexer:** `server indexer` - Runs the indexer component

---

## Current Docker Setup (Before This Project)

### Files
- **Main Dockerfile:** `/home/user/kestra/Dockerfile`
  - Single-stage build
  - Base: `eclipse-temurin:21-jre-jammy`
  - Expects pre-built executable copied into `docker/app/kestra`
  - Build args: `KESTRA_PLUGINS`, `APT_PACKAGES`, `PYTHON_LIBRARIES`

- **PR Dockerfile:** `/home/user/kestra/Dockerfile.pr`
  - Extends `kestra/kestra:$KESTRA_DOCKER_BASE_VERSION`

### Docker Compose Files
- `docker-compose.yml` - Standard setup with Docker socket mount
- `docker-compose-dind.yml` - Docker-in-Docker setup
- `docker-compose-ci.yml` - CI testing databases (MySQL, PostgreSQL)

### Build Process (from Makefile)
```bash
# Build executable JAR
./gradlew writeExecutableJar --no-daemon --parallel

# Copy to docker directory
cp build/executable/* docker/app/kestra && chmod +x docker/app/kestra

# Build Docker image
docker build \
  --compress \
  --rm \
  -f ./Dockerfile \
  --build-arg="APT_PACKAGES=python3 python-is-python3 python3-pip curl jattach" \
  --build-arg="PYTHON_LIBRARIES=kestra" \
  -t ${DOCKER_IMAGE}:${VERSION} ${DOCKER_PATH}
```

---

## Build System Details

### Gradle Build
- **Main Class:** `io.kestra.cli.App` (defined in `build.gradle:53`)
- **Java Version:** 21 (JDK for build, JRE for runtime)
- **Build Tool:** Gradle 8.13 with Shadow plugin
- **Multi-module Project:** 21 modules defined in `settings.gradle`
- **Key Gradle Tasks:**
  - `shadowJar` - Creates fat JAR with all dependencies (depends on `ui:assembleFrontend`)
  - `writeExecutableJar` - Creates executable JAR from shadow JAR
  - `jar` - Standard JAR task
  - `dependencies` - Downloads all dependencies (useful for Docker layer caching)
  - `:ui:assembleFrontend` - Builds frontend using Node.js 22.12.0

### Gradle Module Structure (OSS Repository)
```
kestra (root)
├── platform/             # Bill of Materials (BOM) - centralizes dependency versions
├── model/                # Data models and core annotations (@Plugin, @PluginProperty)
├── processor/            # Annotation processors
├── core/                 # Core engine and functionality
├── tests/                # Shared test utilities
├── storage-local/        # Local filesystem storage
├── repository-memory/    # In-memory repository implementation
├── runner-memory/        # In-memory task runner
├── jdbc/                 # Base JDBC abstraction
├── jdbc-h2/              # H2 database support
├── jdbc-mysql/           # MySQL database support
├── jdbc-postgres/        # PostgreSQL database support
├── script/               # Scripting support
├── executor/             # Task execution engine
├── scheduler/            # Workflow scheduler
├── worker/               # Distributed worker
├── webserver/            # REST API and web backend
├── ui/                   # Frontend UI (React/Vue - requires Node.js)
├── cli/                  # Command-line interface (main entry point)
├── e2e-tests/            # End-to-end tests
└── jmh-benchmarks/       # Performance benchmarks
```

**Note:** Enterprise Edition (EE) modules like `storage-s3`, `storage-gcs`, `storage-minio`, `repository-postgres`, `repository-mysql`, and `runner-kafka` are NOT in the OSS repository.

### Project File Structure
```
/home/user/kestra/
├── docker/               # Docker configuration files
│   ├── app/
│   │   ├── confs/        # Configuration directory
│   │   ├── plugins/      # Plugins directory
│   │   └── secrets/      # Secrets directory
│   └── usr/local/bin/
│       └── docker-entrypoint.sh  # Entrypoint script
├── gradle/               # Gradle wrapper files
│   ├── jar/              # JAR manifest configuration
│   └── wrapper/          # Gradle wrapper distribution
├── build.gradle          # Main build file (plugins, tasks, publishing)
├── settings.gradle       # Gradle module declarations
└── gradle.properties     # Build properties (version, JVM args, caching)
```

### Docker Entrypoint
**File:** `/home/user/kestra/docker/usr/local/bin/docker-entrypoint.sh`
```bash
#!/usr/bin/env sh
set -e
exec /app/kestra "$@"
```
Simple wrapper that executes the Kestra binary with all passed arguments.

---

## Configuration

All Kestra components require:

### Database (Required)
- **PostgreSQL** or **MySQL**
- Used for: Queue, Repository
- Configuration:
  ```yaml
  datasources:
    postgres:
      url: jdbc:postgresql://postgres:5432/kestra
      driverClassName: org.postgresql.Driver
      username: kestra
      password: k3str4
  ```

### Storage (Required)
- **Local**, **S3**, **GCS**, or **MinIO**
- Must be shared/accessible by all components
- Configuration:
  ```yaml
  kestra:
    storage:
      type: local
      local:
        base-path: "/app/storage"
  ```

### Queue (Required)
- **PostgreSQL**, **MySQL**, or **Kafka**
- Configuration:
  ```yaml
  kestra:
    queue:
      type: postgres
  ```

### Repository (Required)
- **PostgreSQL** or **MySQL**
- Configuration:
  ```yaml
  kestra:
    repository:
      type: postgres
  ```

---

## Progress & Deliverables

### ✅ Completed

#### 1. Executor Dockerfile (`Dockerfile.executor`)
- **Multistage build:**
  - **Builder stage:** Compiles from source using Gradle + JDK 21
    - Uses BuildKit cache mounts for Gradle dependencies (`/root/.gradle`, `/build/.gradle`)
    - Mounts `.git` directory (read-only) for version metadata via `gradle-git-properties` plugin
    - Excludes UI build (stub `ui:assembleFrontend` task) to avoid Node.js dependency
    - Skips tests with `-x test` for faster builds
  - **Runtime stage:** Minimal JRE 21 image (`eclipse-temurin:21-jre-jammy`)
- **Build args:** `KESTRA_PLUGINS`, `APT_PACKAGES`, `PYTHON_LIBRARIES`
- **Default command:** `server executor`
- **User:** Non-root `kestra:kestra`
- **Key optimizations:**
  - Layer caching for dependencies (reused across rebuilds)
  - Parallel Gradle builds (`--parallel`)
  - No daemon (`--no-daemon`) for clean builds
  - Stub UI module avoids 5+ minute Node.js build

#### 2. Executor Example (`docker-compose.executor-example.yml`)
- Shows how to build and run executor
- Includes PostgreSQL dependency with healthcheck
- Example environment configuration
- Build args for Python and plugins

#### 3. Executor Documentation (`Dockerfile.executor.md`)
- Build instructions with BuildKit caching
- Configuration requirements (repository, queue, storage)
- Quick test examples (in-memory and PostgreSQL)
- CLI options reference
- Distributed setup requirements
- Multistage build explanation

### ⏳ Remaining Tasks

1. **Dockerfile.worker** - Worker component
2. **Dockerfile.scheduler** - Scheduler component
3. **Dockerfile.webserver** - Webserver/API component
4. **Complete docker-compose example** - All components together in distributed setup
5. **Optional:** Update Makefile with new build targets

---

## Key Files & Locations

### Source Code
| Component | File |
|-----------|------|
| Server CLI | `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/ServerCommand.java` |
| Executor | `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/ExecutorCommand.java` |
| Worker | `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/WorkerCommand.java` |
| Scheduler | `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/SchedulerCommand.java` |
| Webserver | `/home/user/kestra/cli/src/main/java/io/kestra/cli/commands/servers/WebServerCommand.java` |

### Docker Files
| File | Purpose |
|------|---------|
| `/home/user/kestra/Dockerfile` | Original single-stage Dockerfile |
| `/home/user/kestra/Dockerfile.executor` | ✅ Executor multistage Dockerfile |
| `/home/user/kestra/docker-compose.yml` | Standard docker-compose |
| `/home/user/kestra/docker-compose-dind.yml` | Docker-in-Docker setup |
| `/home/user/kestra/docker-compose.executor-example.yml` | ✅ Executor example |

### Build Files
| File | Purpose |
|------|---------|
| `/home/user/kestra/build.gradle` | Main Gradle build configuration |
| `/home/user/kestra/Makefile` | Build automation (includes `build-docker` target) |

---

## Quick Reference Commands

### Build from Source
```bash
# Build executable JAR
./gradlew writeExecutableJar --no-daemon --parallel

# Location of built JAR
ls -lh build/executable/
```

### Docker Build (Original Method)
```bash
# Requires pre-built JAR
make build-docker
```

### Docker Build (New Multistage Method)
```bash
# IMPORTANT: Enable BuildKit for cache support
export DOCKER_BUILDKIT=1

# Build executor (compiles from source)
docker build -f Dockerfile.executor -t kestra/executor:latest .

# With custom packages
docker build -f Dockerfile.executor \
  --build-arg APT_PACKAGES="python3 python-is-python3 curl" \
  --build-arg PYTHON_LIBRARIES="kestra pandas" \
  --build-arg KESTRA_PLUGINS="io.kestra.plugin.aws" \
  -t kestra/executor:latest .
```

**Build Performance:**
- **First build:** ~5-10 minutes (downloads all dependencies)
- **Subsequent builds:** ~1-2 minutes (uses cached Gradle dependencies)
- **No code changes:** Seconds (Docker layer cache)

### Run Components
```bash
# Executor (requires configuration)
docker run --rm \
  -e KESTRA_CONFIGURATION='
kestra:
  repository:
    type: memory
  queue:
    type: memory
  storage:
    type: local
    local:
      base-path: /tmp/kestra-storage
' \
  kestra/executor:latest

# Worker (using original image)
docker run kestra/kestra:latest server worker --thread 128

# Scheduler (using original image)
docker run kestra/kestra:latest server scheduler

# Webserver (using original image)
docker run kestra/kestra:latest server webserver --no-indexer

# Standalone (all-in-one, using original image)
docker run kestra/kestra:latest server standalone
```

**Note:** All components require `kestra.repository.type`, `kestra.queue.type`, and `kestra.storage.type` configuration.

### Git Operations
```bash
# Current branch
git checkout claude/setup-kestra-docker-011CUKtn4M8CaoW8SBpo8uPh

# Status
git status

# Commit (with retry on signing failure)
git add <files>
sleep 2 && git commit -m "message"

# Push
git push -u origin claude/setup-kestra-docker-011CUKtn4M8CaoW8SBpo8uPh
```

---

## Important Notes

### ServerType Enum
Each component sets a `ServerType` via `propertiesOverrides()`:
- `ServerType.EXECUTOR` - ExecutorCommand.java:52
- `ServerType.WORKER` - WorkerCommand.java:34
- `ServerType.SCHEDULER` - SchedulerCommand.java:26
- `ServerType.WEBSERVER` - WebServerCommand.java:54

This affects which beans/services are loaded and started.

### Shared Dependencies
All components need:
- Database connection (PostgreSQL/MySQL)
- Queue configuration
- Storage configuration
- Repository configuration

### Python & Plugins
- Python support via `uv` package manager (version 0.6.17)
- Plugins installed via `/app/kestra plugins install <plugin-list>`
- Custom Python libraries via `uv pip install --system <libraries>`

### Security
- All containers run as non-root `kestra:kestra` user (UID/GID created in Dockerfile)
- Docker socket access requires root (development only, not production)

---

## Next Steps Template

When creating remaining Dockerfiles, follow this pattern:

1. **Read** the command class to understand CLI options
2. **Create** `Dockerfile.<component>` with multistage build
3. **Create** `docker-compose.<component>-example.yml`
4. **Create** `Dockerfile.<component>.md` documentation
5. **Commit** with descriptive message
6. **Push** to branch

### Template Structure
```dockerfile
# Dockerfile.<component>
FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /build
# ... copy source and build ...
RUN ./gradlew writeExecutableJar --no-daemon --parallel

FROM eclipse-temurin:21-jre-jammy
# ... setup user, copy executable, install deps ...
CMD ["server", "<component>"]
```

---

## Environment Variables

### KESTRA_CONFIGURATION
All components accept YAML configuration via this environment variable:

```yaml
datasources:
  postgres:
    url: jdbc:postgresql://postgres:5432/kestra
    driverClassName: org.postgresql.Driver
    username: kestra
    password: k3str4

kestra:
  repository:
    type: postgres
  storage:
    type: local
    local:
      base-path: "/app/storage"
  queue:
    type: postgres
  server:
    basic-auth:
      username: "admin@kestra.io"
      password: kestra
  url: http://localhost:8080/
```

---

## Troubleshooting

### Git Commit Signing Failures
- **Error:** `Service Unavailable` when signing commits
- **Solution:** Retry with exponential backoff (2s delay works)
- **Command:** `sleep 2 && git commit -m "message"`

### Build Failures
- Ensure all source directories are copied to builder stage
- Check Gradle wrapper has execute permissions
- Verify Java 21 is used

### Runtime Issues
- Check database connectivity
- Ensure shared storage is accessible
- Verify queue configuration matches across all components

---

## Lessons Learned & Challenges Overcome

### Challenge 1: Missing COPY Paths in Dockerfile
**Problem:** Original Dockerfile.executor attempted to copy Enterprise Edition modules (`storage-s3`, `storage-gcs`, `storage-minio`, `repository-postgres`, `repository-mysql`, `runner-kafka`) that don't exist in the OSS repository.

**Solution:**
- Identified all modules actually present in `settings.gradle`
- Removed non-existent EE module references
- Added missing OSS modules: `model`, `processor`, `tests`, `script`, `executor`, `scheduler`, `worker`

**Files:** [Dockerfile.executor:19-38](Dockerfile.executor:19-38)

---

### Challenge 2: Build Failure - Missing Git Repository
**Problem:** `gradle-git-properties` plugin failed with "No Git repository found" because `.git` directory wasn't available in Docker build context.

**Error:**
```
FAILURE: Build failed with an exception.
* What went wrong:
Execution failed for task ':generateGitProperties'.
> No Git repository found.
```

**Solution:** Mount `.git` directory read-only during build using BuildKit bind mounts:
```dockerfile
RUN --mount=type=bind,source=.git,target=/build/.git,readonly \
    ./gradlew writeExecutableJar --no-daemon --parallel
```

**Why this works:**
- Provides Git metadata for versioning without copying `.git` into the image
- Keeps image size small and secure
- Generates correct version info from actual Git tags/commits

**Files:** [Dockerfile.executor:21-22, 42-45](Dockerfile.executor:21-22)

---

### Challenge 3: Node.js/npm Build Failure
**Problem:** `shadowJar` task has a hard dependency on `ui:assembleFrontend`, which requires Node.js and npm. The UI build was failing with npm errors, adding 5+ minutes to build time.

**Error:**
```
FAILURE: Build failed with an exception.
* What went wrong:
Execution failed for task ':ui:npmInstall'.
> Process 'command 'npm'' finished with non-zero exit value 1
```

**Initial Failed Approaches:**
1. ❌ Skip task with `-x :ui:assembleFrontend` → "task not found" error
2. ❌ Don't copy UI module → Gradle configuration fails

**Final Solution:** Create a stub `ui/build.gradle` with a no-op `assembleFrontend` task:
```dockerfile
RUN mkdir -p ui/src/main/resources && \
    mkdir -p webserver/src/main/resources/ui && \
    printf 'plugins {\n    id "java"\n}\n\ntasks.register("assembleFrontend") {\n    description = "Stub task - no UI build"\n    doLast {\n        file("../webserver/src/main/resources/ui").mkdirs()\n    }\n}\n' > ui/build.gradle
```

**Why this works:**
- Satisfies Gradle's project structure requirements
- Satisfies `shadowJar.dependsOn 'ui:assembleFrontend'` dependency
- No Node.js download or npm install required
- Executor doesn't need UI anyway (only webserver does)

**Files:** [Dockerfile.executor:13-17](Dockerfile.executor:13-17)

---

### Challenge 4: Slow Rebuild Times
**Problem:** Every Docker build was downloading all Gradle dependencies from scratch (~5-10 minutes), even when nothing changed.

**Solution:** Implement BuildKit cache mounts for Gradle caches:
```dockerfile
# Cache Gradle user home (downloaded dependencies)
RUN --mount=type=cache,target=/root/.gradle \
    # Cache project build cache
    --mount=type=cache,target=/build/.gradle \
    ./gradlew writeExecutableJar --no-daemon --parallel
```

**Results:**
- **Before:** 5-10 minutes per build
- **After (first build):** 5-10 minutes (downloads dependencies once)
- **After (subsequent builds):** 1-2 minutes (only rebuilds changed code)

**Requirement:** Must use `DOCKER_BUILDKIT=1` environment variable

**Files:** [Dockerfile.executor:20-22, 42-45](Dockerfile.executor:20-22)

---

### Challenge 5: Module Dependency Order
**Problem:** Build failed because `core` module depends on annotations from `model` and `processor` modules, but they weren't copied in the correct order.

**Error:**
```
error: package io.kestra.core.models.annotations does not exist
import io.kestra.core.models.annotations.Plugin;
```

**Solution:** Copy modules in dependency order:
1. **Foundation:** `platform` → `model` → `processor`
2. **Core:** `core` → `tests`
3. **Infrastructure:** storage/jdbc/runner modules
4. **Services:** `executor`, `scheduler`, `worker`
5. **API:** `webserver`, `cli`

**Files:** [Dockerfile.executor:20-38](Dockerfile.executor:20-38)

---

### Challenge 6: Configuration Validation on Startup
**Problem:** Docker image runs but immediately fails with configuration errors:

**Error:**
```
ERROR i.k.c.v.ServerCommandValidator Server configuration requires 'kestra.repository.type' to be defined
ERROR i.k.c.v.ServerCommandValidator Server configuration requires 'kestra.queue.type' to be defined
ERROR i.k.c.v.ServerCommandValidator Server configuration requires 'kestra.storage.type' to be defined
```

**Solution:** This is **expected behavior**. All Kestra components require three mandatory configurations via `KESTRA_CONFIGURATION` environment variable:
- `kestra.repository.type` (postgres, mysql, or memory)
- `kestra.queue.type` (postgres, mysql, or memory)
- `kestra.storage.type` (local, s3, gcs, or minio)

**Minimal working configuration:**
```yaml
kestra:
  repository:
    type: memory
  queue:
    type: memory
  storage:
    type: local
    local:
      base-path: /tmp/kestra-storage
```

**Files:** [Dockerfile.executor.md:68-116](Dockerfile.executor.md:68-116)

---

## Key Takeaways

1. **OSS vs EE modules:** Always verify module existence in `settings.gradle` before copying in Dockerfile
2. **Git metadata:** Use BuildKit bind mounts for read-only access to `.git` without bloating image
3. **UI builds:** Executor/scheduler/worker don't need UI - stub it out to save time
4. **BuildKit caching:** Essential for reasonable build times in multi-module Gradle projects
5. **Module dependencies:** Follow dependency graph when copying source files
6. **Configuration validation:** Kestra enforces configuration at startup - this is good architecture

---

**End of Context Document**
