# Technical Plan: Add RabbitMQ (with Management UI) + Persian hands-on guide

Implements the confirmed Logic Plan `docs/__logic/add-rabbitmq.md`.
Scope: orchestration only. No change to `accounting-service`.

## 1. `docker-compose.yml` — new service

Follows the existing naming convention (`s-<name>-service-fastapi` /
`c-<name>-service-fastapi`, volume `fastapi_<name>_data`) and the existing
resource-limit/restart style.

```yaml
  s-rabbitmq-service-fastapi:
    image: rabbitmq:4.1-management
    container_name: c-rabbitmq-service-fastapi
    hostname: rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
      RABBITMQ_DEFAULT_VHOST: ${RABBITMQ_VHOST}
    ports:
      - "${RABBITMQ_MANAGEMENT_PORT:-45683}:15672"
      - "${RABBITMQ_AMQP_PORT:-45684}:5672"
    volumes:
      - fastapi_rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 10s
      retries: 12
      start_period: 30s
    networks:
      - airplus-network
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1024M
        reservations:
          cpus: '0.2'
          memory: 256M
    restart: unless-stopped
```

Plus `fastapi_rabbitmq_data:` under top-level `volumes:`.

Decisions:
- **Image `rabbitmq:4.1-management`** — official image; the `-management`
  variant has the web UI plugin enabled. Pinned to a minor line (4.1), not
  `latest`, so a rebuild doesn't silently jump a major version.
- **`hostname: rabbitmq`** — RabbitMQ stores its data under a directory named
  after the node name (`rabbit@<hostname>`). Without a fixed hostname, each
  new container gets a random hostname and "loses" the data in the volume.
- **`RABBITMQ_DEFAULT_*`** — on first boot (empty volume) the image creates
  this administrator user and this vhost (instead of `guest` and `/`).
  Only applied on first boot; see guide §troubleshooting for changing them
  later.
- **Ports** — `45683` = management UI (browser), `45684` = AMQP (apps/tools
  on the host), continuing the existing `4568x` range. Both overridable from
  `.env`. Inside the Docker network, services always use
  `s-rabbitmq-service-fastapi:5672`.
- **Healthcheck** — `rabbitmq-diagnostics -q ping`; `start_period` covers the
  Erlang VM boot time. No service `depends_on` it yet (accounting-service
  isn't wired to RabbitMQ in this task), so it can't block the stack.
- **No `depends_on` added to `s-accounting-service-fastapi`** — out of scope
  (logic plan).

## 2. `.env` — new keys

```
RABBITMQ_HOST=s-rabbitmq-service-fastapi
RABBITMQ_PORT=5672
RABBITMQ_USER=airplus
RABBITMQ_PASSWORD=airplus-dev-only-change-me   # dev-only; .env is committed to a public repo
RABBITMQ_VHOST=accounting
RABBITMQ_MANAGEMENT_PORT=45683
RABBITMQ_AMQP_PORT=45684
```

`RABBITMQ_HOST`/`RABBITMQ_PORT` are the in-network address future services
will read (the accounting service already loads this `.env` via
`env_file`); unused for now. `.env` stays committed, matching current
practice — moving it out of git is a separate task (logic plan side note).

## 3. `docs/rabbitmq-guide.md` — Persian guide (new)

Sections (per logic plan):
1. Start / stop / status commands, panel URL and login.
2. Core concepts with examples from our future flow: producer, exchange
   (direct/topic/fanout), queue (classic vs quorum), binding + routing key,
   consumer, ack/nack, durable + persistent, dead-letter exchange, vhost.
3. Mapping to the future flow: document engine → ledger → reply
   (illustrative only).
4. Panel tour: Overview, Connections, Channels, Exchanges, Queues and
   Streams, Admin.
5. Exercise A — panel only: in vhost `accounting` create a DLX + dead-letter
   queue, a topic exchange, an inbox queue (with DLX argument) and a replies
   queue, bindings, publish from the panel, "Get messages" with ack / reject
   (→ dead-letter), and watch the message-rate charts. All objects prefixed
   `demo.` and a cleanup step at the end.
6. Exercise B — same topology from the command line via the bundled
   `rabbitmqadmin` (v2) run with `docker compose exec`, plus the HTTP API
   with `curl` (what apps/tools do under the hood).
7. Troubleshooting: port busy, `guest` login refused, credentials changed in
   `.env` after first boot, messages gone after restart (non-durable queue /
   non-persistent message), message published but "unroutable".

## 4. `CLAUDE.md` — Commands section

Add the panel/AMQP URLs next to the existing service URLs, a
`docker compose logs -f s-rabbitmq-service-fastapi` line, and a pointer to
`docs/rabbitmq-guide.md`.

## 5. Verification

- `docker compose config` validates (with the external network present).
- `docker compose up -d s-rabbitmq-service-fastapi` → container becomes
  `healthy`.
- `curl -u $RABBITMQ_USER:$RABBITMQ_PASSWORD http://localhost:45683/api/overview`
  returns 200; `/api/vhosts` lists `accounting`; `guest` login is refused.
- Run every command in guide Exercise B end-to-end (declare → publish → get →
  dead-letter → cleanup) against the running container.
- Restart the container and confirm a durable queue + persistent message
  survive.
