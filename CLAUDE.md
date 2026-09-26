# CLAUDE.md

> This is the DevOps/orchestration project. It owns `docker-compose.yml` and
> the shared `.env`, and wires together the sibling service projects that
> live alongside this directory. It contains no application/business logic.

---

## 1. Project Overview

- **Purpose:** orchestrate all AirPlus services (currently `accounting-service`;
  future services are added here, not inside their own repos).
- **Expectation:** every service referenced in `docker-compose.yml` must be
  checked out as a sibling directory next to `devops/`, e.g.
  `../accounting-service`.

---

## 2. Git Workflow

- **Never commit directly to `main`.** Before starting any task (feature, fix, chore), create and switch to a new branch first.
- Branch naming: `<type>/<short-task-description>`, e.g. `feat/add-redis`, `fix/env-var-typo`, `chore/rename-service`.
- If already on `main` when a task begins, create the branch before making any file changes — don't code first and branch later.
- One task = one branch. Don't stack unrelated work onto an existing task branch.

---

## 3. Task Planning Workflow (mandatory, before any change)

Every task (feature, fix, chore) goes through two plan documents, in order,
**before any file is added or modified**:

1. **Logic Plan (Persian, no code)** — write a plain-language plan in
   **Persian (Farsi)** describing the scenario, purpose, and use cases (e.g.
   why a new service is being added, how it relates to the others). No
   code, no exact file/service names — this is for human/business review.
   Save it as a new file under `docs/__logic/`, e.g.
   `docs/__logic/<task-slug>.md`.
2. **Wait for confirmation.** Do not proceed to the technical plan or to any
   change until the user has reviewed and confirmed the logic plan.
3. **Technical Plan (English, detailed)** — only after the logic plan is
   confirmed, write a detailed technical plan in **English** covering every
   file to be added or changed (`docker-compose.yml` services, `.env` keys,
   networks, volumes). Save it as a new file under `docs/__plan/`, e.g.
   `docs/__plan/<task-slug>.md`.
4. Only after the technical plan exists may implementation begin, and it
   should follow the branch rule in Section 2.

---

## 4. Repository Layout

```
airplus/
├── devops/                    # you are here
│   ├── docker-compose.yml      # orchestrates all services
│   ├── .env                    # shared env vars (DB creds, ports, etc.)
│   ├── docs/
│   │   ├── __logic/             # per-task Logic Plans (Persian, no code)
│   │   └── __plan/              # per-task Technical Plans (English, detailed)
│   └── CLAUDE.md
├── accounting-service/         # sibling service — FastAPI backend, its own repo
└── <future-service>/           # future sibling services get wired in here
```

Each service keeps its own Dockerfile inside its own repo; this repo only
references it via a relative `build:` path (e.g. `../accounting-service`).

---

## 5. Never Do

- Commit or push directly to `main` — always work on a task branch (see Section 2)
- Write or modify `docker-compose.yml`/`.env` before the Logic Plan and Technical Plan exist and the Logic Plan has been confirmed (see Section 3)
- Put application/business logic in this repo — it belongs in the service's own repo
- Reference a service by an absolute path — always use a relative sibling path (`../<service-name>`) so this repo works regardless of where the parent folder is cloned
- Add a service's Dockerfile here — it stays in the service's own repo

---

## 6. Commands

```bash
# Build & run all services
docker compose build
docker compose up -d
docker compose logs -f s-accounting-service-fastapi

# accounting-service: http://localhost:45680  (docs at /docs)
# mysql:               localhost:45681
# phpMyAdmin:          http://localhost:45682
# RabbitMQ panel:      http://localhost:45683  (login: RABBITMQ_USER / RABBITMQ_PASSWORD from .env)
# RabbitMQ AMQP:       localhost:45684 from the host; s-rabbitmq-service-fastapi:5672 inside the network
# RabbitMQ guide (Persian, hands-on): docs/rabbitmq-guide.md
docker compose logs -f s-rabbitmq-service-fastapi

# Messaging workers (outbox relay + consumer) — same image as the API, no ports
docker compose logs -f s-messaging-relay-fastapi s-messaging-consumer-fastapi
# they do NOT auto-reload: restart them after changing accounting-service code
docker compose restart s-messaging-relay-fastapi s-messaging-consumer-fastapi

# Migrations (run inside the accounting-service container)
docker compose exec s-accounting-service-fastapi alembic revision --autogenerate -m "<message>"
docker compose exec s-accounting-service-fastapi alembic upgrade head

# Seed initial data
docker compose exec s-accounting-service-fastapi python seed.py
```
