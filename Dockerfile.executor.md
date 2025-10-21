# Kestra Executor Dockerfile

This Dockerfile builds a dedicated image for running the Kestra Executor component in a distributed setup.

## What is the Executor?

The Executor is the Kestra component responsible for processing and executing workflow tasks. In a distributed Kestra deployment, you can run multiple executors to scale your task execution capacity.

## Building the Image

### Using Docker Build (with caching for faster builds)

The Dockerfile uses BuildKit cache mounts to speed up rebuilds by caching Gradle dependencies:

```bash
# Enable BuildKit for cache support
export DOCKER_BUILDKIT=1

# Build with cache (much faster on subsequent builds)
docker build -f Dockerfile.executor -t kestra/executor:latest .
```

**First build**: Downloads all dependencies (~5-10 minutes)
**Subsequent builds**: Reuses cached dependencies (only rebuilds changed code)

### With Build Arguments

You can customize the build with the following arguments:

```bash
docker build -f Dockerfile.executor \
  --build-arg APT_PACKAGES="python3 python-is-python3 python3-pip curl" \
  --build-arg PYTHON_LIBRARIES="kestra pandas numpy" \
  --build-arg KESTRA_PLUGINS="io.kestra.plugin.aws io.kestra.plugin.gcp" \
  -t kestra/executor:latest .
```

### Advanced: Using External Gradle Cache

To share the Gradle cache across different builds or CI/CD pipelines, you can use a named volume:

```bash
# Create a named volume for the Gradle cache
docker volume create gradle-cache

# Build using the named volume
docker build -f Dockerfile.executor \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t kestra/executor:latest .
```

The cache is automatically managed by Docker's BuildKit and persists between builds.

### Using Docker Compose

```bash
docker compose -f docker-compose.executor-example.yml build
docker compose -f docker-compose.executor-example.yml up
```

## Running the Executor

The executor requires three mandatory configurations:
- **`kestra.repository.type`** - Where flow definitions are stored (postgres, mysql, or memory)
- **`kestra.queue.type`** - Where task queues are stored (postgres, mysql, or memory)
- **`kestra.storage.type`** - Where execution outputs are stored (local, s3, gcs, minio)

### Quick Test (In-Memory - Development Only)

For quick testing without external dependencies:

```bash
docker run --rm \
  --name kestra-executor \
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
```

**Note**: Memory configuration loses all data when the container stops. Use for testing only!

### Production (PostgreSQL)

```bash
docker run -d \
  --name kestra-executor \
  --network kestra-network \
  -e KESTRA_CONFIGURATION='
datasources:
  postgres:
    url: jdbc:postgresql://postgres:5432/kestra
    driverClassName: org.postgresql.Driver
    username: kestra
    password: k3str4
kestra:
  repository:
    type: postgres
  queue:
    type: postgres
  storage:
    type: local
    local:
      base-path: /app/storage
' \
  -v kestra-storage:/app/storage \
  kestra/executor:latest
```

### With Docker Compose

See `docker-compose.executor-example.yml` for a complete example.

## Configuration

The executor requires access to:
- **Database**: PostgreSQL or MySQL for queue and repository
- **Storage**: Shared storage accessible by all Kestra components (local, S3, GCS, MinIO)

### Environment Variables

Configure the executor using the `KESTRA_CONFIGURATION` environment variable with YAML configuration:

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
  queue:
    type: postgres
  executor:
    thread-count: 128  # Number of threads for task execution
```

## Distributed Setup

In a production distributed setup, you'll need:

1. **Scheduler** (Dockerfile.scheduler) - Schedules workflows
2. **Executor** (Dockerfile.executor) - Executes tasks (this component)
3. **Worker** (Dockerfile.worker) - Processes worker tasks
4. **Webserver** (Dockerfile.webserver) - Provides UI and API

All components must share the same database and storage configuration.

## Multistage Build Details

This Dockerfile uses a multistage build:

1. **Builder Stage**: Compiles Kestra from source using Gradle
   - Base: `eclipse-temurin:21-jdk-jammy`
   - Builds the executable JAR

2. **Runtime Stage**: Creates minimal runtime image
   - Base: `eclipse-temurin:21-jre-jammy`
   - Copies only the executable and necessary dependencies
   - Runs as non-root user `kestra`

## Command Line Options

The executor supports additional command-line options:

```bash
docker run kestra/executor:latest server executor --help
```

Common options:
- `--skip-executions`: Skip specific execution IDs (for troubleshooting)
- `--skip-flows`: Skip specific flows (for troubleshooting)
- `--skip-namespaces`: Skip specific namespaces (for troubleshooting)

See ExecutorCommand.java:31-41 for the full list of options.
