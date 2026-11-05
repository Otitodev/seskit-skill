# Installing SESKit

Docs: <https://otitodev.github.io/seskit/getting-started/installation/>

## What gets started

You do not install PostgreSQL or Redis. Compose starts all of it.

| Service | What it is |
|---|---|
| `db` | PostgreSQL 16 |
| `redis` | Redis 7. The job queue, auth cache and rate limits |
| `mailpit` | Catches local mail, so no AWS account is needed to try it |
| `migrate` | Runs once and exits: `alembic upgrade head`. `api` and `worker` wait for it |
| `api` | `/v1` and the dashboard, port 8000 |
| `worker` | Sends mail, delivers webhooks, polls for events. **Not optional** |

A failed migration stops the stack. That is deliberate.

Data lives in named Docker volumes. `docker compose down` keeps it.
**`docker compose down -v` deletes it.** Never run `-v` on an instance with data.

Postgres and Redis are on host ports **55432** and **56379**, not 5432 and
6379, because machines with PostgreSQL installed often already use the usual
ones. Inside Compose they are `db:5432` and `redis:6379`.

## Before you start

Check Docker and the Compose plugin are installed and running:

```bash
docker --version && docker compose version && docker info > /dev/null
```

If not, tell the user. Do not install them.

## Option 1: try it locally, no AWS

```bash
git clone https://github.com/Otitodev/seskit.git && cd seskit
cp .env.example .env
docker compose up -d --wait
docker compose exec api python -m seskit_api.doctor
```

`.env.example` already sets `SECRET_KEY`, `DATABASE_URL` and `REDIS_URL`
correctly for Compose. `ENVIRONMENT=local` allows the placeholder secret.

## Option 2: a real server

The installer does the same thing without the placeholder secret:

```bash
curl -fsSL https://otitodev.github.io/seskit/install.sh | sh
```

Clones the repo, writes `.env` with a generated `SECRET_KEY`, starts the stack.
It refuses rather than guesses: missing Docker means it stops and says so. It is
safe to run twice and never overwrites an existing `.env`.

It installs the **latest release**, not `main`. Options:

| Variable | Default |
|---|---|
| `SESKIT_VERSION` | Latest release. A tag like `v0.1.0`, or `main` |
| `SESKIT_DIR` | `./seskit` |
| `SESKIT_PUBLIC_URL` | Guessed from the machine's public IP |

If the user prefers not to pipe to a shell, do it by hand:

```bash
git clone https://github.com/Otitodev/seskit.git && cd seskit
cp .env.example .env
# Replace SECRET_KEY=changeme with a long random value, without printing it:
#   openssl rand -base64 32
docker compose up -d --wait
```

Outside `ENVIRONMENT=local`, SESKit refuses to boot with the example
`SECRET_KEY`.

## Check it

```bash
docker compose exec api python -m seskit_api.doctor
```

Seven checks: secret key, database URL, Postgres reachable and at the right
schema, Redis, which path a send would take, and whether anything is draining
the queue. Exit code 0 or 1. Every check should pass, except possibly the
public-URL one on a laptop.

## The owner account

The user opens the dashboard and registers. **The first registration claims the
instance and signup closes behind it.**

Right after registering, SESKit shows a recovery code like `K7QD-4M2X-9PHB-TWE6`,
**once**. The user must save it. It is the way back in if they forget their
password, with no email and no shell.

## Before real mail on a server

The shipped `docker-compose.yml` is a development stack. For a server:

- Put a **TLS terminator** in front (Caddy, nginx or a load balancer). Port 8000
  is plain HTTP.
- Behind a proxy that terminates TLS, set `FORWARDED_ALLOW_IPS=*`, or the
  session cookie loses its `Secure` flag and redirects can point at `http://`.
- Set `PUBLIC_BASE_URL` (for example `https://mail.example.com`) in `.env`. It
  puts a working one-click unsubscribe link on every message. Unset, messages go
  out with no unsubscribe header at all. Both processes read it.
- Remove the Mailpit service and the source bind mounts, and do not publish
  Postgres and Redis to the host. (They are bound to `127.0.0.1` by default.)
- API and worker must share the **same** `SECRET_KEY`, `DATABASE_URL` and
  `REDIS_URL`. A mismatched `SECRET_KEY` fails silently: every unsubscribe link
  says "not valid" while everything else works.

Full guide: <https://otitodev.github.io/seskit/operating/deploying/>

## No Docker

Only for someone working on SESKit or already running Postgres and Redis.
Requires `uv`.

```bash
uv sync
docker compose up -d db redis mailpit        # or point at your own
export DATABASE_URL="postgresql+asyncpg://seskit:seskit@localhost:55432/seskit"
export REDIS_URL="redis://localhost:56379/0"
export SECRET_KEY="dev"
uv run alembic upgrade head
uv run uvicorn seskit_api.main:app --reload
uv run arq seskit_worker.main.WorkerSettings    # second shell, same environment
```

## On a platform with no Compose (Render, Fly, Railway, Aeroplane)

Two services from the same repo: the API from `docker/Dockerfile`, the worker
from `docker/Dockerfile.worker`. Give both the same `SECRET_KEY`,
`DATABASE_URL` and `REDIS_URL`. Run `alembic upgrade head` as a release command,
or set `MIGRATE_ON_START=true` on the API if the platform has no release hook.
Only the API needs a public port.

## Next

AWS and real sending: [aws.md](aws.md). First message: [sending.md](sending.md).
