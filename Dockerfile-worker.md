# Kestra Worker Dockerfile

This Dockerfile builds a dedicated image for running the Kestra Worker component in a distributed setup.

## What is the Worker?

The Worker is the Kestra component responsible for processing worker tasks - tasks that require isolation or specific resources. In a distributed Kestra deployment, you can run multiple workers with different configurations, resource limits, or worker groups.

## Building the Image

### Using Docker Build (with caching for faster builds)

The Dockerfile uses BuildKit cache mounts to speed up rebuilds by caching Gradle dependencies:

```bash
# Enable BuildKit for cache support
export DOCKER_BUILDKIT=1

# Build with cache (much faster on subsequent builds)
docker build -f Dockerfile.worker -t kestra/worker:latest .
```

**First build**: Downloads all dependencies (~5-10 minutes)
**Subsequent builds**: Reuses cached dependencies (only rebuilds changed code)

### With Build Arguments

You can customize the build with the following arguments:

```bash
docker build -f Dockerfile.worker \
  --build-arg APT_PACKAGES="python3 python-is-python3 python3-pip curl" \
  --build-arg PYTHON_LIBRARIES="kestra pandas numpy" \
  --build-arg KESTRA_PLUGINS="io.kestra.plugin.aws io.kestra.plugin.gcp" \
  -t kestra/worker:latest .
```

### Advanced: Using External Gradle Cache

To share the Gradle cache across different builds or CI/CD pipelines, you can use a named volume:

```bash
# Create a named volume for the Gradle cache
docker volume create gradle-cache

# Build using the named volume
docker build -f Dockerfile.worker \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t kestra/worker:latest .
```

The cache is automatically managed by Docker's BuildKit and persists between builds.

### Using Docker Compose

```bash
docker compose -f docker-compose.worker-example.yml build
docker compose -f docker-compose.worker-example.yml up
```

## Running the Worker

The worker requires three mandatory configurations:
- **`kestra.repository.type`** - Where flow definitions are stored (postgres, mysql, or memory)
- **`kestra.queue.type`** - Where task queues are stored (postgres, mysql, or memory)
- **`kestra.storage.type`** - Where execution outputs are stored (local, s3, gcs, minio)

### Quick Test (In-Memory - Development Only)

For quick testing without external dependencies:

```bash
docker run --rm \
  --name kestra-worker \
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
  kestra/worker:latest
```

**Note**: Memory configuration loses all data when the container stops. Use for testing only!

### Production (PostgreSQL)

```bash
docker run -d \
  --name kestra-worker \
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
  kestra/worker:latest
```

### With Docker Compose

See `docker-compose.worker-example.yml` for a complete example with executor.

## Configuration

The worker requires access to:
- **Database**: PostgreSQL or MySQL for queue and repository
- **Storage**: Shared storage accessible by all Kestra components (local, S3, GCS, MinIO)
- **Executor**: Must be running to process tasks from the worker queue

### Environment Variables

Configure the worker using the `KESTRA_CONFIGURATION` environment variable with YAML configuration:

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
  storage:
    type: local
    local:
      base-path: /app/storage
```

## Worker-Specific Options

### Thread Count

Control the maximum number of worker threads (defaults to 8 × CPU cores):

```bash
docker run kestra/worker:latest server worker --thread 128
```

Or via docker-compose:

```yaml
worker:
  command: ["server", "worker", "--thread", "128"]
```

### Worker Groups (Enterprise Edition Only)

Assign the worker to a specific worker group for task routing:

```bash
docker run kestra/worker:latest server worker --worker-group my-group
```

**Requirements:**
- Worker group key must match pattern: `[a-zA-Z0-9_-]+`
- Only available in Kestra Enterprise Edition

Or via docker-compose:

```yaml
worker:
  command: ["server", "worker", "--worker-group", "gpu-workers"]
```

**Use cases for worker groups:**
- GPU-enabled workers for ML tasks
- High-memory workers for data processing
- Workers with specific tools or dependencies
- Geographic distribution of workers

## Distributed Setup

In a production distributed setup, you'll need:

1. **Scheduler** (Dockerfile.scheduler) - Schedules workflows
2. **Executor** (Dockerfile.executor) - Executes tasks
3. **Worker** (Dockerfile.worker) - Processes worker tasks (this component)
4. **Webserver** (Dockerfile.webserver) - Provides UI and API

All components must share the same database and storage configuration.

### Component Interaction

```
┌─────────────┐      ┌──────────┐      ┌────────┐
│  Scheduler  │─────▶│  Queue   │◀─────│Executor│
└─────────────┘      └──────────┘      └────────┘
                           │                │
                           │                │
                           ▼                ▼
                     ┌──────────┐    ┌──────────┐
                     │  Worker  │    │  Tasks   │
                     │  Queue   │    │          │
                     └──────────┘    └──────────┘
                           │
                           ▼
                     ┌──────────┐
                     │  Worker  │ (this component)
                     └──────────┘
```

The worker:
1. Polls the worker queue for tasks
2. Executes tasks in isolated threads
3. Reports results back to the queue
4. Executor picks up results and continues flow execution

## Multistage Build Details

This Dockerfile uses a multistage build:

1. **Builder Stage**: Compiles Kestra from source using Gradle
   - Base: `eclipse-temurin:21-jdk-jammy`
   - Builds the executable JAR
   - Uses BuildKit cache for Gradle dependencies
   - Skips UI build (not needed for worker)

2. **Runtime Stage**: Creates minimal runtime image
   - Base: `eclipse-temurin:21-jre-jammy`
   - Copies only the executable and necessary dependencies
   - Runs as non-root user `kestra`
   - Installs Python support via `uv`
   - Optionally installs Kestra plugins

## Command Line Options

The worker supports the following command-line options:

```bash
docker run kestra/worker:latest server worker --help
```

Common options:
- `-t, --thread <count>` - Max number of worker threads (default: 8 × CPU cores)
- `-g, --worker-group <key>` - Worker group key (EE only, pattern: `[a-zA-Z0-9_-]+`)

See [WorkerCommand.java:25-29](cli/src/main/java/io/kestra/cli/commands/servers/WorkerCommand.java:25-29) for implementation details.

## Scaling Workers

You can scale workers horizontally to increase task processing capacity:

### Docker Compose

```bash
# Scale to 3 worker instances
docker compose -f docker-compose.worker-example.yml up --scale worker=3
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kestra-worker
spec:
  replicas: 3  # Scale to 3 workers
  selector:
    matchLabels:
      app: kestra-worker
  template:
    metadata:
      labels:
        app: kestra-worker
    spec:
      containers:
      - name: worker
        image: kestra/worker:latest
        env:
        - name: KESTRA_CONFIGURATION
          value: |
            # ... your configuration ...
```

## Resource Requirements

Typical resource allocation per worker:

- **CPU**: 2-4 cores (depends on thread count and task types)
- **Memory**: 2-8 GB (depends on task complexity)
- **Disk**: Minimal (uses shared storage)

Adjust based on your workload characteristics.

## Troubleshooting

### Worker not picking up tasks

1. Check that executor is running
2. Verify database connectivity
3. Ensure queue configuration matches across all components
4. Check worker logs for errors

### Worker group tasks not running

1. Verify you're using Enterprise Edition
2. Check worker group key format (must match `[a-zA-Z0-9_-]+`)
3. Ensure tasks are assigned to the correct worker group

### Out of memory errors

1. Increase worker memory allocation
2. Reduce thread count with `--thread` option
3. Use worker groups to isolate resource-intensive tasks

## Best Practices

1. **Thread count**: Start with default (8 × CPU cores) and tune based on metrics
2. **Worker groups**: Use for isolation of different workload types (GPU, memory, etc.)
3. **Horizontal scaling**: Add more worker instances rather than increasing threads
4. **Resource limits**: Set Docker memory/CPU limits to prevent resource exhaustion
5. **Shared storage**: Ensure all workers can access the same storage backend
6. **Monitoring**: Track worker queue depth and task execution times

## Examples

### High-throughput setup

```yaml
worker:
  image: kestra/worker:latest
  command: ["server", "worker", "--thread", "256"]
  deploy:
    replicas: 5
  resources:
    limits:
      cpus: '8'
      memory: 16G
```

### GPU worker group (EE)

```yaml
worker-gpu:
  image: kestra/worker:latest
  command: ["server", "worker", "--worker-group", "gpu", "--thread", "4"]
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

## See Also

- [Dockerfile.executor.md](Dockerfile.executor.md) - Executor component documentation
- [docker-compose.worker-example.yml](docker-compose.worker-example.yml) - Complete example setup
- [Kestra Worker Documentation](https://kestra.io/docs/architecture#worker)
