# Technical Plan: intake reconciler worker container

Implements `docs/__logic/intake-reconciler-worker.md`. The worker code is
`app/workers.py`'s `reconcile-intake` subcommand in accounting-service
(`docs/__plan/stuck-intake-recovery.md` §5 there, already merged).

## `docker-compose.yml`

New service, same shared settings as the existing two workers
(`docs/__plan/messaging-workers.md`): `image`/`build` from `../accounting-service`,
`env_file: .env`, bind mount `../accounting-service:/app`, network `airplus-network`, no ports,
`depends_on` mysql **and** rabbitmq with `condition: service_healthy`, `restart: unless-stopped`,
`PYTHONUNBUFFERED: "1"`, `stop_grace_period: 30s`, limits `cpus: '0.5'` / `memory: 256M`,
reservations `0.1` / `64M`.

| Service | Container | `command` |
|---|---|---|
| `s-intake-reconciler-fastapi` | `c-intake-reconciler-fastapi` | `python -m app.workers reconcile-intake` |

## Fix alongside this (found while wiring the above)

`s-messaging-consumer-fastapi`'s `command` was hardcoded to `consume ledger diagnostics` —
correct when `docs/__plan/messaging-workers.md` was written (only those two services existed), but
the accounting-service has since registered a `document` service (and `intake_test`) that this
container was never consuming. This is the actual root cause of the incident that led to the
`stuck-intake-recovery` task: the ledger's replies to the document engine were being sent but never
picked up. Fix: drop the explicit service list so it runs `python -m app.workers consume` (documented
default: every registered service — `app/workers.py`'s own `--help`), so a future new service is
never silently unconsumed again.

## Verification

- `docker compose config -q` passes.
- Full stack up locally: all three worker containers start after mysql and rabbitmq report healthy.
- A document sent through the document engine's "درخواست ثبت در دفترداری" gets a reply and its
  status updates (the consumer fix); a document manually left in `posting` gets picked up by the
  reconciler on its next pass (the new container).
