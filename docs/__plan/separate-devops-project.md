# Technical Plan: Separate DevOps Project from `accounting-service`

Confirms and implements the confirmed Logic Plan at
`docs/__logic/separate-devops-project.md`, with the naming decision:
the API project directory is **`accounting-service`** (not `api`), and
docker-compose service/container names use `accounting-service` instead of
`financial`.

## Final directory layout

```
/home/malekipourdev/airplus/
├── accounting-service/        # renamed from centeral-accounting-system — git history preserved
│   ├── .git/
│   ├── .claude/
│   ├── .vscode/
│   ├── alembic/
│   ├── alembic.ini
│   ├── app/
│   ├── database-diagram/
│   ├── docs/
│   │   ├── __logic/            # logic plans for accounting-service tasks
│   │   └── __plan/             # technical plans for accounting-service tasks
│   ├── Dockerfile              # stays — builds THIS service's image only
│   ├── requirements.txt
│   └── CLAUDE.md                # updated: drop docker-compose/.env ownership
└── devops/                    # NEW repo, `git init` only (no remote yet)
    ├── .git/
    ├── docker-compose.yml      # moved from accounting-service, service names updated
    ├── .env                    # moved from accounting-service
    ├── docs/
    │   ├── __logic/
    │   └── __plan/
    │       └── separate-devops-project.md   # this same plan, copied here once devops repo exists
    └── CLAUDE.md                # new, devops-focused
```

## Steps

### 1. Rename/move the current repo (preserves git history)

```bash
mv /home/malekipourdev/airplus/centeral-accounting-system \
   /home/malekipourdev/airplus/accounting-service
```

> ⚠️ This changes the Claude Code session's working directory out from
> under it. Requires explicit go-ahead and likely a restart/reopen of the
> session at the new path afterward.

### 2. Create the new `devops` repo

```bash
mkdir /home/malekipourdev/airplus/devops
cd /home/malekipourdev/airplus/devops
git init
mkdir -p docs/__logic docs/__plan
```

### 3. Move orchestration files out of `accounting-service` into `devops`

```bash
cd /home/malekipourdev/airplus/accounting-service
git mv docker-compose.yml /home/malekipourdev/airplus/devops/docker-compose.yml
git mv .env /home/malekipourdev/airplus/devops/.env
git commit -m "chore: move docker-compose.yml and .env to devops project"
```

(`Dockerfile` stays in `accounting-service` — it builds this service's own
image, per the existing "Never Do" rule.)

### 4. Update `devops/docker-compose.yml`

- Rename service key `s-financial-service-fastapi` → `s-accounting-service-fastapi`
- Rename `container_name: c-financial-service-fastapi` → `c-accounting-service-fastapi`
- `mysql`/`phpmyadmin` service and container names are unchanged (they don't
  reference "financial").
- Fix the build context and bind mount, since `docker-compose.yml` no longer
  sits next to the service code (sibling-repo layout):
  - `build: .` → `build: ../accounting-service`
  - `volumes: - .:/app` → `volumes: - ../accounting-service:/app`
- `env_file: - .env` is unchanged (still sits next to `docker-compose.yml`
  in `devops/`).

Resulting relevant block:

```yaml
services:
  s-accounting-service-fastapi:
    build: ../accounting-service
    container_name: c-accounting-service-fastapi
    ports:
      - "45680:8000"
    env_file:
      - .env
    volumes:
      - ../accounting-service:/app
    depends_on:
      s-mysql-service-fastapi:
        condition: service_healthy
    networks:
      - airplus-network
    ...
```

### 5. Update `accounting-service/CLAUDE.md`

- Remove `docker-compose.yml` / `.env` from the repository layout diagram
  and from "Commands" (those now live in the `devops` repo).
- Remove the "Never Do" line about Dockerfile placement referencing root
  `docker-compose.yml` ownership — rephrase to note that orchestration is
  owned by the sibling `devops` repo.
- Keep everything else (task planning workflow, architecture rules, module
  checklist) as-is — this repo is still governed by the same process.

### 6. Create `devops/CLAUDE.md` (new file)

A slim CLAUDE.md scoped to orchestration only:
- Purpose: owns `docker-compose.yml`, shared `.env`, and wiring for all
  sibling service projects (currently just `accounting-service`; future
  services are added here).
- Same Git Workflow rule (Section 2 from the current CLAUDE.md: never
  commit to `main` directly, branch per task).
- Same Task Planning Workflow rule (Section 3): Logic Plan (Persian) →
  confirm → Technical Plan (English) → implement.
- Expects sibling checkouts: any service referenced in `docker-compose.yml`
  must exist as a sibling directory (e.g. `../accounting-service`).
- Commands section moved from the old CLAUDE.md (`docker compose build`,
  `up -d`, `logs -f`, migrations via `docker compose exec`, etc.), now run
  from inside `devops/`.

### 7. Copy this plan pair into the `devops` repo

Since this task's outcome (moving `docker-compose.yml`) belongs to the
`devops` repo going forward, copy (not move — keep the record in
`accounting-service` too, since the decision originated there):
- `docs/__logic/separate-devops-project.md` → `devops/docs/__logic/`
- `docs/__plan/separate-devops-project.md` → `devops/docs/__plan/`

### 8. Commit in both repos

- `accounting-service`: commit the CLAUDE.md update, the plan docs, and the
  `git mv` removals (steps 3 & 5 & 7's copy) on branch
  `chore/separate-devops-project` (already checked out), then open a PR.
- `devops`: first commit on its `main`/`master` branch — no PR (no remote
  yet), just an initial local commit.

## Out of scope for this task

- No remote (GitHub/GitLab) is created for `devops` — local `git init` only,
  per your answer.
- No change to `database-diagram/`, `alembic/`, or any application code.
- No `.gitignore` added for `.env` — it's already tracked today; changing
  that would be a separate decision.
