# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

An operational (no build/test) Docker Compose setup that runs **Microsoft SQL Server 2017** in a single container, with documented backup/restore workflows. There is no application code — the deliverables are `docker-compose.yaml`, `.env`, and the backup/restore procedures in `README.md` (written in Thai).

## Architecture

- One service, `MSSQL_SERVER` (container name `mssql17`), built from `mcr.microsoft.com/mssql/server:2017-latest`. Runs as `user: root`, `restart: always`, with a `sqlcmd SELECT 1` healthcheck.
- All configuration is injected from `.env` via `${VAR}` substitution — the compose file hardcodes nothing. `.env` is gitignored, so it must exist locally before any `docker compose` command works.
- **The data lives entirely in an external named volume `mssql_data`** mounted at `/var/opt/mssql`. `external: true` means Compose will NOT create it — `docker volume create mssql_data` must be run once before first `up`, or the container fails to start. Destroying the container never touches the data; the volume is the single source of truth.
- Because the volume is external and named, host-OS tools can't read it directly. Every backup/restore in the README goes through a throwaway `ubuntu` container that mounts the volume — that indirection is the whole point of the procedures, not incidental.

## Common operations

```bash
docker volume create mssql_data          # once, before first run (volume is external)
docker compose up -d                     # start
docker compose down                      # stop (data survives in the volume)
docker logs mssql17 --tail 20            # troubleshoot startup
docker ps                                # confirm port 1433 is mapped
```

Backup the volume to `backup/mssql_volume.tar.gz` (works on any OS):

```bash
docker run --rm -v mssql_data:/data -v "$(pwd)/backup":/backup ubuntu \
  tar -czvf /backup/mssql_volume.tar.gz -C /data .
```

Restore: `docker compose down` first, then a temp `ubuntu` container clears the volume, `docker cp`s the tarball in, and extracts it — full step-by-step is in `README.md` under "🔄 Restore". Always stop SQL Server before restoring, and start it again after.

## Conventions and cautions

- `.tar.gz` and `.env` are gitignored on purpose. Do not commit database dumps or secrets — a large `mssql_volume.tar.gz` may exist in the working tree but must stay untracked.
- Git flow: work lands on `develop`, releases go through `release/*` branches merged to `main` and tagged (`1.0.x`). Version numbers appear in both the README title and git tags — keep them in sync when bumping.
- SA password must satisfy MSSQL policy (8+ chars, uppercase + digit + special) or the container silently fails to initialize.
