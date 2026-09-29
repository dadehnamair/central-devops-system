# Technical Plan: Reduce MySQL memory usage (disable performance_schema)

Implements the confirmed Logic Plan `docs/__logic/tune-mysql-memory.md`.
Scope: `s-mysql-service-fastapi` only. No change to `accounting-service`, `.env`,
ports, volumes, networks, or resource limits.

## 1. Baseline (measured 2026-09-29)

| Item | Value |
|---|---|
| Image | `mysql:8.0` (8.0.44) |
| Container memory | ~506 MiB |
| Data size (`appdb`) | ~4 MB |
| `performance_schema` | ON |
| `innodb_buffer_pool_size` | 128M (default) |
| Host swap used | 1839 / 2047 MiB |

`performance_schema` accounts for most of the container's memory and is not
used by this project (no monitoring stack reads it).

## 2. `docker-compose.yml` — `s-mysql-service-fastapi`

Add an explicit `command` (the image default is `mysqld`, so this only appends
one flag):

```yaml
  s-mysql-service-fastapi:
    image: mysql:8.0
    container_name: c-mysql-service-fastapi
    command: ["mysqld", "--performance-schema=OFF"]
    ...
```

A command-line flag is used instead of a mounted `conf.d` file to keep the
change self-contained in the compose file (no new files/mounts).

`innodb_buffer_pool_size` is intentionally left at the 128M default: it is
~30x the current data size, and lowering it would save <64 MiB while risking
performance as `appdb` grows.

## 3. Rollout

```bash
docker compose config -q                                        # validate
docker compose up -d --no-deps s-mysql-service-fastapi          # recreate MySQL only
```

`--no-deps` ensures only the MySQL container is recreated. The data volume
`fastapi_mysql_data` is reused, so no data is touched.

Expected downtime: until the healthcheck passes (typically 10–30 s).

## 4. Dependents

`s-accounting-service-fastapi`, `s-messaging-relay-fastapi`,
`s-messaging-consumer-fastapi` and `s-phpmyadmin-service-fastapi` are not
recreated. The accounting service's SQLAlchemy engines use
`pool_pre_ping=True`, so stale connections are replaced automatically.
Messages are durable in RabbitMQ, so anything not processed during the outage
is retried.

If a dependent does not recover, restart it:

```bash
docker compose restart s-messaging-relay-fastapi s-messaging-consumer-fastapi
```

## 5. Verification

1. `docker compose ps` — MySQL is `healthy`.
2. `SELECT @@performance_schema;` → `0`; `@@innodb_buffer_pool_size` unchanged (128M).
3. `docker stats` — container memory well below the ~506 MiB baseline.
4. `curl http://localhost:45680/docs` → `200`.
5. Relay/consumer logs show no repeating DB errors after the restart.
6. `appdb` table count matches the pre-change count.

## 6. Rollback

Remove the `command:` line and run
`docker compose up -d --no-deps s-mysql-service-fastapi`.
