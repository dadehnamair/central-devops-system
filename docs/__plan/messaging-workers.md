# Technical Plan: messaging worker containers

Implements `docs/__logic/messaging-workers.md`. The worker code is
`app/workers.py` in accounting-service (`docs/__plan/messaging-infrastructure.md` §9, §13 there).

## `docker-compose.yml`

- `s-accounting-service-fastapi`: add `image: airplus/accounting-service:dev`, so the
  workers reuse the exact image the API builds (one build, three containers).
- Two new services. Both share these settings:
  - `image: airplus/accounting-service:dev`, `build: ../accounting-service`
  - `env_file: .env`, bind mount `../accounting-service:/app`, network `airplus-network`
  - no ports
  - `depends_on` mysql **and** rabbitmq with `condition: service_healthy`
  - `restart: unless-stopped`
  - limits `cpus: '0.5'`, `memory: 256M`; reservations `0.1` / `64M`

| Service | Container | `command` |
|---|---|---|
| `s-messaging-relay-fastapi` | `c-messaging-relay-fastapi` | `python -m app.workers relay` |
| `s-messaging-consumer-fastapi` | `c-messaging-consumer-fastapi` | `python -m app.workers consume ledger diagnostics` |

- `PYTHONUNBUFFERED: "1"` in both, so logs appear immediately in `docker compose logs`.
- `stop_grace_period: 30s`, so SIGTERM lets the in-flight message finish (the workers handle SIGTERM).

## `.env`

Optional tuning keys, with the code's defaults. They are listed so they are discoverable:
- `MESSAGING_RELAY_POLL_INTERVAL_MS=500`
- `MESSAGING_RELAY_BATCH_SIZE=100`
- `MESSAGING_RELAY_MAX_ATTEMPTS=10`
- `MESSAGING_RETRY_MAX_ATTEMPTS=3`
- `MESSAGING_RETRY_BASE_DELAY_MS=2000`

## `CLAUDE.md` / `docs/rabbitmq-guide.md`

- Commands: worker logs, and a restart after a code change.
- Guide: a short section on the real queues the system now creates (`acc.*` exchanges; `ledger.commands`, `diagnostics.replies` with `.retry`/`.dlq`) and the test console's «پیام‌رسانی» tab.

## Verification

- `docker compose config -q` passes.
- Full stack up locally: the workers start after mysql and rabbitmq report healthy, and connect.
- A ping sent from the API through the stack gets a reply.
