# PayTrace Deployment

`paytrace-deployment` contains deployment scripts and runtime composition files for PayTrace services.

The current Docker Compose setup starts the PayTrace file ingest worker from the Docker Hub image `openfintechlab/paytrace-file-ingest:latest`. The worker watches the shared CSV directory, reads configuration from `PAYTRACE_ENV_FILE`, publishes payment rows to RabbitMQ, and records processing state in PostgreSQL.

## Compose Files

```text
compose/
  docker-compose.yml
```

The Compose project currently defines these services:

- `paytrace-file-ingest` - CSV file ingest worker using `openfintechlab/paytrace-file-ingest:latest`
- `paytrace-postgres` - PostgreSQL database used by the ingest worker
- `paytrace-rabbitmq` - RabbitMQ broker used for payment row dispatch

## Prerequisites

- Docker Engine with Docker Compose v2
- Existing environment file, for example `config/.env` or `../paytrace-file-ingest-csv/.env`
- Existing `fwcsv/` workspace with the expected ingest folders

PostgreSQL and RabbitMQ are defined in `compose/docker-compose.yml`. If you change the Compose file to use host-managed PostgreSQL or RabbitMQ instead, remember that `localhost` inside a container means the container itself. On Docker Desktop for macOS and Windows, use `host.docker.internal` when a container must reach a service running on the host.

## Environment File

Set the file-ingest environment file before running any Compose command from this project root.

PowerShell, using this repository's `config/.env`:

```powershell
$env:PAYTRACE_ENV_FILE=(Resolve-Path .\config\.env).Path
```

PowerShell, using the sibling `paytrace-file-ingest-csv/.env` file:

```powershell
$env:PAYTRACE_ENV_FILE=(Resolve-Path ..\paytrace-file-ingest-csv\.env).Path
```

Bash, using this repository's `config/.env`:

```bash
export PAYTRACE_ENV_FILE="$(realpath ./config/.env)"
```

Bash, using the sibling `paytrace-file-ingest-csv/.env` file:

```bash
export PAYTRACE_ENV_FILE="$(realpath ../paytrace-file-ingest-csv/.env)"
```

Compose uses this value for the `paytrace-file-ingest` service `env_file`. Using an absolute path avoids ambiguity because relative `env_file` paths are resolved from `compose/docker-compose.yml`, not the current shell directory.

The same environment file must also be passed through `--env-file` so Compose can interpolate `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `RABBITMQ_DEFAULT_USER`, and `RABBITMQ_DEFAULT_PASS` before creating containers.

## CSV Workspace Directory

By default, Compose mounts `../../fwcsv` into the ingest container as `/app/fwcsv`. Override the host-side directory with `PAYTRACE_FWCSV_HOST_DIR` before running Compose.

PowerShell:

```powershell
$env:PAYTRACE_FWCSV_HOST_DIR=(Resolve-Path ..\fwcsv).Path
```

Bash:

```bash
export PAYTRACE_FWCSV_HOST_DIR="$(realpath ../fwcsv)"
```

For a different ingest workspace, point the variable at that folder:

```powershell
$env:PAYTRACE_FWCSV_HOST_DIR="C:\Users\baqai\paytrace-fwcsv-test"
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml up -d paytrace-file-ingest
```

`PAYTRACE_FWCSV_HOST_DIR` is read by Docker Compose while it renders the bind mount. Set it in the shell, in the file passed with `--env-file`, or in a Compose project `.env` file. Service-level `env_file` values alone are not enough for volume interpolation.

## Validate Configuration

From this project root:

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml config --quiet
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml config --quiet
```

This validates the Compose file, confirms the referenced environment file can be loaded, and supplies the variables used by Compose interpolation for PostgreSQL and RabbitMQ.

## Start Services

Start all services in the background.

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml up -d
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml up -d
```

Start or recreate only the ingest worker.

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml up -d paytrace-file-ingest
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml up -d paytrace-file-ingest
```

Pull the latest image before starting.

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml pull paytrace-file-ingest
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml up -d paytrace-file-ingest
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml pull paytrace-file-ingest
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml up -d paytrace-file-ingest
```

## PostgreSQL Initialization

The SQL folder is mounted into PostgreSQL and into a one-shot init service as `/docker-entrypoint-initdb.d`:

```yaml
- ../sql:/docker-entrypoint-initdb.d:ro
```

Because `compose/docker-compose.yml` lives in the `compose/` directory, `../sql` resolves to this repository's `sql/` directory. The service `paytrace-postgres-init` runs `sql/01_create_paytrace_ingest.sql` after PostgreSQL is healthy, then `paytrace-file-ingest` starts only after that init service completes successfully.

The SQL script creates tables under the `paytrace_ingest` schema. The Compose file overrides the ingest worker to use that schema:

```yaml
OFTL_POSTGRESDB_SCHEMA: paytrace_ingest
```

Recreate the database from scratch only when you intentionally want to delete the existing PostgreSQL volume and all data in it.

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml down -v
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml up -d
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml down -v
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml up -d
```

Run the SQL manually without deleting the existing volume.

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml up -d paytrace-postgres
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml exec paytrace-postgres psql -U admin -d paytrace -f /docker-entrypoint-initdb.d/01_create_paytrace_ingest.sql
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml up -d paytrace-postgres
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml exec paytrace-postgres psql -U admin -d paytrace -f /docker-entrypoint-initdb.d/01_create_paytrace_ingest.sql
```

## Check Status And Logs

Show service status:

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml ps
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml ps
```

Follow worker logs:

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml logs -f paytrace-file-ingest
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml logs -f paytrace-file-ingest
```

Follow PostgreSQL logs:

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml logs -f paytrace-postgres
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml logs -f paytrace-postgres
```

## Stop Services

Stop the worker without removing containers:

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml stop paytrace-file-ingest
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml stop paytrace-file-ingest
```

Stop and remove Compose-managed containers and networks:

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml down
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml down
```

## Restart Services

Restart the ingest worker:

PowerShell:

```powershell
docker compose --env-file .\config\.env -f .\compose\docker-compose.yml restart paytrace-file-ingest
```

Bash:

```bash
docker compose --env-file ./config/.env -f ./compose/docker-compose.yml restart paytrace-file-ingest
```

## Runtime Notes

- Set `PAYTRACE_ENV_FILE` before running Compose commands. Compose fails fast if this variable is missing.
- Prefer an absolute `PAYTRACE_ENV_FILE` value, for example PowerShell's `(Resolve-Path .\config\.env).Path` or Bash's `$(realpath ./config/.env)`.
- Pass `--env-file .\config\.env` on Windows or `--env-file ./config/.env` on Linux and macOS so Compose can interpolate PostgreSQL and RabbitMQ settings before it starts containers.
- The ingest service loads runtime environment variables from `PAYTRACE_ENV_FILE` through `env_file`.
- `OFTL_POSTGRESDB_SCHEMA` should match the schema created by the SQL script. The included script creates `paytrace_ingest`.
- `OFTL_FWCSV_ROOTDIR` is overridden to `/app/fwcsv` inside the container.
- `OFTL_RABITMQ_HOST` is overridden to `paytrace-rabbitmq` inside the ingest container.
- `PAYTRACE_FWCSV_HOST_DIR` controls the host directory mounted to `/app/fwcsv`; when omitted, Compose uses `../../fwcsv` relative to `compose/docker-compose.yml`.
- The service uses `restart: unless-stopped`, so Docker restarts it after failures or daemon restarts unless it was manually stopped.
