# PayTrace Deployment

`paytrace-deployment` contains deployment scripts and runtime composition files for PayTrace services.

The current Docker Compose setup starts the PayTrace file ingest worker from the Docker Hub image `openfintechlab/paytrace-file-ingest:latest`. The worker watches the shared CSV directory, reads configuration from `PAYTRACE_ENV_FILE`, publishes payment rows to RabbitMQ, and records processing state in PostgreSQL.

## Compose Files

```text
compose/
  docker-compose.yml
```

The Compose project currently defines this service:

- `paytrace-file-ingest` - CSV file ingest worker using `openfintechlab/paytrace-file-ingest:latest`

## Prerequisites

- Docker Engine with Docker Compose v2
- Existing `paytrace-file-ingest-csv/.env`
- Reachable PostgreSQL and RabbitMQ instances
- Existing `fwcsv/` workspace with the expected ingest folders

If PostgreSQL or RabbitMQ run on the host machine, remember that `localhost` inside the container means the container itself. On Docker Desktop for macOS, use `host.docker.internal` in the service `.env` file when the container must reach host services.

## Environment File

Set the file-ingest environment file before running any Compose command from this project root:

```bash
export PAYTRACE_ENV_FILE=../../paytrace-file-ingest-csv/.env
```

Compose uses this value for the `paytrace-file-ingest` service `env_file`. The path is relative to `compose/docker-compose.yml`, not the current shell directory.

## Validate Configuration

From this project root:

```bash
docker compose -f compose/docker-compose.yml config --quiet
```

This validates the Compose file and confirms the referenced environment file can be loaded.

## Start Services

Start the file ingest worker in the background:

```bash
docker compose -f compose/docker-compose.yml up -d
```

Start or recreate only the ingest worker:

```bash
docker compose -f compose/docker-compose.yml up -d paytrace-file-ingest
```

Pull the latest image before starting:

```bash
docker compose -f compose/docker-compose.yml pull paytrace-file-ingest
docker compose -f compose/docker-compose.yml up -d paytrace-file-ingest
```

## Check Status And Logs

Show service status:

```bash
docker compose -f compose/docker-compose.yml ps
```

Follow worker logs:

```bash
docker compose -f compose/docker-compose.yml logs -f paytrace-file-ingest
```

## Stop Services

Stop the worker without removing containers:

```bash
docker compose -f compose/docker-compose.yml stop paytrace-file-ingest
```

Stop and remove Compose-managed containers and networks:

```bash
docker compose -f compose/docker-compose.yml down
```

## Restart Services

Restart the ingest worker:

```bash
docker compose -f compose/docker-compose.yml restart paytrace-file-ingest
```

## Runtime Notes

- Set `PAYTRACE_ENV_FILE` before running Compose commands. Compose fails fast if this variable is missing.
- Use `PAYTRACE_ENV_FILE={LOCATION OF .env}/.env` when running Compose commands from this repository root because `env_file` paths are resolved relative to `compose/docker-compose.yml`.
- The ingest service loads runtime environment variables from `PAYTRACE_ENV_FILE` through `env_file`.
- `OFTL_FWCSV_ROOTDIR` is overridden to `/app/fwcsv` inside the container.
- `OFTL_RABITMQ_HOST` is overridden to `paytrace-rabbitmq` inside the ingest container.
- The local workspace `../fwcsv` is mounted to `/app/fwcsv`.
- The service uses `restart: unless-stopped`, so Docker restarts it after failures or daemon restarts unless it was manually stopped.
