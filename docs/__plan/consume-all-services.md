# Technical Plan: ignore `.env`

Implements `docs/__logic/consume-all-services.md`.

- `.gitignore` — ignore `.env`, so the local secrets file is never committed again.

The consumer command change originally planned here landed on `main` in PR #4,
and `LEDGER_DIRECT_ENTRY_MODE` was dropped because the accounting service no
longer reads it (ledger posted-only model).
