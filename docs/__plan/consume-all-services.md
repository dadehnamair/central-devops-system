# Technical Plan: consume every messaging service; direct-entry switch key

Implements `docs/__logic/consume-all-services.md`.

1. `docker-compose.yml` — `s-messaging-consumer-fastapi.command` becomes
   `["python", "-m", "app.workers", "consume"]`. The accounting service's
   worker CLI treats "no names" as every service in `app/composition.py`,
   which now includes `intake_test`.
2. `.env.example` — add `LEDGER_DIRECT_ENTRY_MODE=enabled`
   (`enabled` / `console_only` / `disabled`), read by the accounting API.
3. `.gitignore` — ignore `.env`.

Verification: `docker compose config` is valid; the consumer's startup log
lists `ledger.commands, diagnostics.replies, intake_test.replies,
intake_test.events`.

Deploy order: merge the accounting-service change first. An older image
rejects an empty service list, so the consumer would crash-loop until the
image is rebuilt.
