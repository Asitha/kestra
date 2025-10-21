# Kestra Scheduler Dockerfile

This Dockerfile builds a dedicated image for running the Kestra Scheduler component in a distributed setup.

## What is the Scheduler?

The Scheduler is the Kestra component responsible for scheduling workflows based on triggers (cron schedules, event triggers, etc.). It monitors flow definitions for triggers and queues executions when trigger conditions are met. In a distributed Kestra deployment, you typically run **one scheduler instance** (though it can be scaled for high availability).

## Building the Image

### Using Docker Build (with caching for faster builds)

The Dockerfile uses BuildKit cache mounts to speed up rebuilds by caching Gradle dependencies:

```bash
# Enable BuildKit for cache support
export DOCKER_BUILDKIT=1

# Build with cache (much faster on subsequent builds)
docker build -f Dockerfile.scheduler -t kestra/scheduler:latest .
```

**First build**: Downloads all dependencies (~5-10 minutes)
**Subsequent builds**: Reuses cached dependencies (only rebuilds changed code)

### With Build Arguments

You can customize the build with the following arguments:

```bash
docker build -f Dockerfile.scheduler \
  --build-arg APT_PACKAGES="python3 python-is-python3 python3-pip curl" \
  --build-arg PYTHON_LIBRARIES="kestra pandas numpy" \
  --build-arg KESTRA_PLUGINS="io.kestra.plugin.aws io.kestra.plugin.gcp" \
  -t kestra/scheduler:latest .
```

### Using Docker Compose

```bash
docker compose -f docker-compose.scheduler-example.yml build
docker compose -f docker-compose.scheduler-example.yml up
```

## Running the Scheduler

The scheduler requires three mandatory configurations:
- **`kestra.repository.type`** - Where flow definitions are stored (postgres, mysql, or memory)
- **`kestra.queue.type`** - Where task queues are stored (postgres, mysql, or memory)
- **`kestra.storage.type`** - Where execution outputs are stored (local, s3, gcs, minio)

### Quick Test (In-Memory - Development Only)

For quick testing without external dependencies:

```bash
docker run --rm \
  --name kestra-scheduler \
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
  kestra/scheduler:latest
```

**Note**: Memory configuration loses all data when the container stops. Use for testing only!

### Production (PostgreSQL)

```bash
docker run -d \
  --name kestra-scheduler \
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
  kestra/scheduler:latest
```

### With Docker Compose

See `docker-compose.scheduler-example.yml` for a complete example with executor and worker.

## Configuration

The scheduler requires access to:
- **Database**: PostgreSQL or MySQL for queue and repository
- **Storage**: Shared storage accessible by all Kestra components (local, S3, GCS, MinIO)
- **Executor & Worker**: Must be running to process scheduled executions

### Environment Variables

Configure the scheduler using the `KESTRA_CONFIGURATION` environment variable with YAML configuration:

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

## Scheduler Behavior

### How It Works

```
┌─────────────────┐
│   Scheduler     │
│                 │
│  1. Monitor     │───▶ Repository (flow definitions with triggers)
│     flows with  │
│     triggers    │
│                 │
│  2. Evaluate    │───▶ Check cron expressions, event conditions
│     trigger     │
│     conditions  │
│                 │
│  3. Queue       │───▶ Queue (create executions when triggered)
│     executions  │
└─────────────────┘
         │
         ▼
   ┌──────────┐
   │ Executor │ (picks up executions from queue)
   └──────────┘
```

The scheduler:
1. **Monitors**: Continuously scans flow definitions for triggers
2. **Evaluates**: Checks trigger conditions (cron schedules, events, etc.)
3. **Queues**: Creates executions in the queue when triggers fire
4. **Does NOT execute**: Actual task execution is handled by executor/worker

### Trigger Types

The scheduler handles these trigger types:

- **Schedule** - Cron-based scheduling (`0 0 * * *` for daily at midnight)
- **Flow** - Triggered when another flow completes
- **Polling** - Polls external systems for changes
- **Webhook** - Handles webhook events (requires webserver)

## Distributed Setup

In a production distributed setup, you'll need:

1. **Scheduler** (Dockerfile.scheduler) - Schedules workflows (this component)
2. **Executor** (Dockerfile.executor) - Executes tasks
3. **Worker** (Dockerfile.worker) - Processes worker tasks
4. **Webserver** (Dockerfile.webserver) - Provides UI and API

All components must share the same database and storage configuration.

### Component Interaction

```
┌───────────┐
│ Scheduler │──▶ Monitors flows with triggers
└───────────┘
      │
      ▼ (creates executions)
┌───────────┐
│   Queue   │
└───────────┘
      │
      ▼ (picks up executions)
┌───────────┐      ┌────────┐
│ Executor  │─────▶│ Tasks  │
└───────────┘      └────────┘
                        │
                        ▼ (worker tasks)
                  ┌────────┐
                  │ Worker │
                  └────────┘
```

## Multistage Build Details

This Dockerfile uses a multistage build:

1. **Builder Stage**: Compiles Kestra from source using Gradle
   - Base: `eclipse-temurin:21-jdk-jammy`
   - Builds the executable JAR
   - Uses BuildKit cache for Gradle dependencies
   - Skips UI build (not needed for scheduler)

2. **Runtime Stage**: Creates minimal runtime image
   - Base: `eclipse-temurin:21-jre-jammy`
   - Copies only the executable and necessary dependencies
   - Runs as non-root user `kestra`
   - Installs Python support via `uv`
   - Optionally installs Kestra plugins

## Command Line Options

The scheduler has **no command-line options** - it's configured entirely through environment variables.

```bash
docker run kestra/scheduler:latest server scheduler
```

See [SchedulerCommand.java:24-28](cli/src/main/java/io/kestra/cli/commands/servers/SchedulerCommand.java:24-28) for implementation details.

## High Availability

### Single Instance (Recommended for Most Cases)

For most deployments, **one scheduler instance is sufficient**:

```yaml
scheduler:
  image: kestra/scheduler:latest
  deploy:
    replicas: 1  # Single instance
```

**Why single instance?**
- Scheduler is lightweight and can handle thousands of flows
- Multiple schedulers would create duplicate executions
- Database provides persistence if scheduler restarts

### Multiple Instances (Advanced)

For high availability, you can run multiple schedulers with leader election:

```yaml
scheduler:
  image: kestra/scheduler:latest
  deploy:
    replicas: 2  # Multiple instances with leader election
```

**How it works:**
- One scheduler acts as leader, others are standby
- If leader fails, another scheduler takes over
- Requires database-based leader election (built-in)

## Resource Requirements

Typical resource allocation for scheduler:

- **CPU**: 1-2 cores (scheduler is I/O bound, not CPU intensive)
- **Memory**: 1-2 GB (depends on number of flows with triggers)
- **Disk**: Minimal (uses shared storage)

Adjust based on:
- Number of flows with triggers
- Trigger evaluation frequency
- Trigger complexity (polling triggers may need more resources)

## Troubleshooting

### Scheduler not creating executions

1. Check that flows have valid triggers defined
2. Verify database connectivity
3. Ensure queue configuration matches across all components
4. Check scheduler logs for trigger evaluation errors

### Triggers firing multiple times

1. Verify only one scheduler is running (or leader election is working)
2. Check for duplicate flow definitions in repository
3. Review trigger configuration (cron expressions, conditions)

### Missed schedules

1. Check scheduler uptime and restarts
2. Verify system clock is accurate
3. Review trigger backfill configuration
4. Check for resource constraints (CPU, memory)

## Best Practices

1. **Single instance**: Run one scheduler unless you need HA
2. **Monitoring**: Track trigger evaluation metrics and execution creation
3. **Clock sync**: Ensure system clock is accurate for cron triggers
4. **Resource limits**: Set memory limits to prevent OOM
5. **Shared storage**: Ensure all components can access the same storage
6. **Logging**: Enable debug logging for trigger evaluation if needed

## Examples

### Basic setup (single scheduler)

```yaml
scheduler:
  image: kestra/scheduler:latest
  environment:
    KESTRA_CONFIGURATION: |
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
  resources:
    limits:
      cpus: '2'
      memory: 2G
```

### High availability setup

```yaml
scheduler:
  image: kestra/scheduler:latest
  deploy:
    replicas: 2  # Leader election enabled automatically
    restart_policy:
      condition: on-failure
      max_attempts: 3
  environment:
    KESTRA_CONFIGURATION: |
      # ... same configuration ...
```

## Common Trigger Examples

### Daily schedule

```yaml
triggers:
  - id: daily
    type: io.kestra.core.models.triggers.types.Schedule
    cron: "0 0 * * *"  # Every day at midnight
```

### Hourly schedule

```yaml
triggers:
  - id: hourly
    type: io.kestra.core.models.triggers.types.Schedule
    cron: "0 * * * *"  # Every hour
```

### Flow trigger

```yaml
triggers:
  - id: after-etl
    type: io.kestra.core.models.triggers.types.Flow
    inputs:
      flow: etl-job
      namespace: production
```

## See Also

- [Dockerfile.executor.md](Dockerfile.executor.md) - Executor component documentation
- [Dockerfile.worker.md](Dockerfile.worker.md) - Worker component documentation
- [docker-compose.scheduler-example.yml](docker-compose.scheduler-example.yml) - Complete example setup
- [Kestra Scheduler Documentation](https://kestra.io/docs/architecture#scheduler)
- [Trigger Documentation](https://kestra.io/docs/workflow-components/triggers)
