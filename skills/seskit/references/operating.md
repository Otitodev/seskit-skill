# Operating a running instance

Docs: <https://otitodev.github.io/seskit/operating/upgrading/>

## Health

```bash
curl -fsS "$URL/readyz"                                   # 200: Postgres and Redis reachable
docker compose exec api python -m seskit_api.doctor        # seven checks, each failure names the fix
docker compose ps
docker compose logs -f api worker
```

Use `/readyz`, not `/healthz`. `/healthz` returns 200 while the database is
unreachable. Logs hold ids and statuses only, never bodies or recipients.

## Upgrading

Back up **first**, every time:

```bash
docker compose exec db pg_dump -U seskit seskit > backup.sql
```

Then, in the checkout:

```bash
git fetch --tags
git checkout <release-tag>           # a release, not the tip of main
docker compose up -d --build --wait
```

Under Compose the one-shot `migrate` service applies migrations **before** the
new `api` and `worker` start, and a failed migration stops the stack. Migrations
run first because the database is written to be one step ahead of the code.

To read a migration before running it, see
<https://otitodev.github.io/seskit/operating/upgrading/>.

After upgrading:

- `GET /readyz` returns 200
- the worker is running, and a test message leaves `queued`
- if delivery events are set up, the worker is still consuming the queue

**Downgrades are not a recovery plan.** A downgrade that drops a column destroys
its data. Restore from the backup. There is no zero-downtime story: plan a short
maintenance window.

If the instance was installed with the one-line installer, it is a normal git
checkout in `./seskit` (or `$SESKIT_DIR`), and the same steps apply.

## Backup and restore

`pg_dump` as above. The dump does not contain `SECRET_KEY`, and **without the
same `SECRET_KEY`, restored AWS keys cannot be decrypted**, so every project must
be reconnected. Keep the secret somewhere safe, but not stored beside the dump.
Detail:
<https://otitodev.github.io/seskit/operating/backup/>.

## Rotating SECRET_KEY

Don't, unless you must. It disconnects every project: stored AWS keys cannot be
read, and sending fails with a message saying so. Have the AWS access keys to
hand first, rotate, then reconnect each project on the AWS page. The API and
worker must always share the same value.

## Troubleshooting

Run the doctor first. Then, in order of how often each is the answer:

**Mail stuck at `queued`.** The worker is not running.
`docker compose ps`, then `docker compose logs -f worker`. If it is running but
nothing moves, check Redis: a worker that cannot reach it says so once at
startup and then sits quietly.

**`domain_not_verified` / unverified sender.** The `from` address is not covered
by a verified identity and AWS is connected. Add and verify it on the **Domains**
page ([aws.md](aws.md#4-verify-a-sender)). SESKit refuses rather than falling
back to Mailpit.

**Rejected recipient.** The AWS account is probably still in the SES sandbox:
recipients must be verified. [aws.md](aws.md#5-the-ses-sandbox).

**No delivery events, or dashes instead of rates.** Delivery events are not set
up, or the worker is not consuming the queue. Check the AWS page and the worker
logs.

**Webhooks not arriving, or signature fails.** Webhooks need delivery events
first. For signatures, verify against the **raw body bytes** with
`{timestamp}.{body}`, using the endpoint's own signing secret.

**Every unsubscribe link says "not valid".** API and worker have different
`SECRET_KEY` values. Nothing logs an error.

**No unsubscribe header on messages.** `PUBLIC_BASE_URL` is unset on the worker.

**Will not start.**

| Symptom | Cause |
|---|---|
| Complains about `SECRET_KEY` | Still the example value outside `ENVIRONMENT=local` |
| Cannot reach the database | Check `DATABASE_URL`; on a laptop the host port is 55432 |
| Connects but the data is wrong | Something else is bound to that port |
| Migrations fail | Compare `alembic current` against `alembic history` |

**Redirects go to `http://` or the login cookie is not `Secure`.** Behind a
TLS-terminating proxy, set `FORWARDED_ALLOW_IPS=*`.

All of it: <https://otitodev.github.io/seskit/operating/troubleshooting/>.

## Forgotten dashboard password

There is no emailed reset link.

1. **Recovery code.** On the sign-in page choose **Forgot your password?**, then
   give the email, the code shown at sign-up, and a new password (12+
   characters). The code works once and a new one is shown: save it. It signs
   the account out everywhere.
2. **No code.** Set `RECOVERY_KEY` (at least 24 random characters) in the
   environment, redeploy, and give it in the same box in place of the code.
   Remove it afterwards.
3. **With a shell.** `docker compose exec api seskit reset-password`. It prompts
   for the email and a hidden new password.

Ask the user before setting `RECOVERY_KEY` or resetting a password. Never print
it back.

## Receiving mail, suppression, metrics

Not covered here. Start from the docs index:
<https://otitodev.github.io/seskit/llms.txt>.
