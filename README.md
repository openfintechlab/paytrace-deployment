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

Set the file-ingest environment file before running any Compose command from this project root.

PowerShell:

```powershell
$env:PAYTRACE_ENV_FILE=(Resolve-Path ..\paytrace-file-ingest-csv\.env).Path
```

Bash:

```bash
export PAYTRACE_ENV_FILE="$(realpath ../paytrace-file-ingest-csv/.env)"
```

Compose uses this value for the `paytrace-file-ingest` service `env_file`. Using an absolute path avoids ambiguity because relative `env_file` paths are resolved from `compose/docker-compose.yml`, not the current shell directory.

## Validate Configuration

From this project root:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml config --quiet
```

This validates the Compose file, confirms the referenced environment file can be loaded, and supplies the variables used by Compose interpolation for PostgreSQL and RabbitMQ.

## Start Services

Start the file ingest worker in the background:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml up -d
```

Start or recreate only the ingest worker:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml up -d paytrace-file-ingest
```

Pull the latest image before starting:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml pull paytrace-file-ingest
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml up -d paytrace-file-ingest
```

## Check Status And Logs

Show service status:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml ps
```

Follow worker logs:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml logs -f paytrace-file-ingest
```

## Stop Services

Stop the worker without removing containers:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml stop paytrace-file-ingest
```

Stop and remove Compose-managed containers and networks:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml down
```

## Restart Services

Restart the ingest worker:

```powershell
docker compose --env-file ..\paytrace-file-ingest-csv\.env -f compose/docker-compose.yml restart paytrace-file-ingest
```

## Runtime Notes

- Set `PAYTRACE_ENV_FILE` before running Compose commands. Compose fails fast if this variable is missing.
- Prefer an absolute `PAYTRACE_ENV_FILE` value, for example PowerShell's `(Resolve-Path ..\paytrace-file-ingest-csv\.env).Path`.
- Pass `--env-file ..\paytrace-file-ingest-csv\.env` so Compose can interpolate PostgreSQL and RabbitMQ settings before it starts containers.
- The ingest service loads runtime environment variables from `PAYTRACE_ENV_FILE` through `env_file`.
- `OFTL_FWCSV_ROOTDIR` is overridden to `/app/fwcsv` inside the container.
- `OFTL_RABITMQ_HOST` is overridden to `paytrace-rabbitmq` inside the ingest container.
- The local workspace `../fwcsv` is mounted to `/app/fwcsv`.
- The service uses `restart: unless-stopped`, so Docker restarts it after failures or daemon restarts unless it was manually stopped.
