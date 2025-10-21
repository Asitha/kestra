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
- **Build Tool:** Gradle with Shadow plugin
- **Key Gradle Tasks:**
  - `shadowJar` - Creates fat JAR with all dependencies
  - `writeExecutableJar` - Creates executable JAR from shadow JAR
  - `jar` - Standard JAR task

### Project Structure
```
/home/user/kestra/
├── cli/                  # CLI module
├── core/                 # Core module
├── jdbc*/                # JDBC modules (h2, mysql, postgres)
├── repository-*/         # Repository implementations
├── runner-*/             # Runner implementations (kafka, memory)
├── storage-*/            # Storage implementations (local, s3, gcs, minio)
├── webserver/            # Webserver module
├── ui/                   # Frontend UI
├── platform/             # Platform module
├── docker/               # Docker configuration files
│   ├── app/
│   │   ├── confs/        # Configuration directory
│   │   ├── plugins/      # Plugins directory
│   │   └── secrets/      # Secrets directory
│   └── usr/local/bin/
│       └── docker-entrypoint.sh  # Entrypoint script
├── build.gradle          # Main build file
└── settings.gradle       # Gradle settings
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
  - **Runtime stage:** Minimal JRE 21 image
- **Build args:** `KESTRA_PLUGINS`, `APT_PACKAGES`, `PYTHON_LIBRARIES`
- **Default command:** `server executor`
- **User:** Non-root `kestra:kestra`

#### 2. Executor Example (`docker-compose.executor-example.yml`)
- Shows how to build and run executor
- Includes PostgreSQL dependency
- Example environment configuration

#### 3. Executor Documentation (`Dockerfile.executor.md`)
- Build instructions
- Configuration examples
- CLI options reference
- Distributed setup requirements

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
# Build executor (compiles from source)
docker build -f Dockerfile.executor -t kestra/executor:latest .

# With custom packages
docker build -f Dockerfile.executor \
  --build-arg APT_PACKAGES="python3 python-is-python3 curl" \
  --build-arg PYTHON_LIBRARIES="kestra pandas" \
  --build-arg KESTRA_PLUGINS="io.kestra.plugin.aws" \
  -t kestra/executor:latest .
```

### Run Components
```bash
# Executor
docker run kestra/executor:latest server executor

# Worker
docker run kestra/kestra:latest server worker --thread 128

# Scheduler
docker run kestra/kestra:latest server scheduler

# Webserver
docker run kestra/kestra:latest server webserver --no-indexer

# Standalone (all-in-one)
docker run kestra/kestra:latest server standalone
```

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

**End of Context Document**
