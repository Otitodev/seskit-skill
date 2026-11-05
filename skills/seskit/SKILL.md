---
name: seskit
description: Set up, connect, use and upgrade SESKit, the self-hosted email platform built on Amazon SES. Use when the user wants to install or deploy SESKit from scratch, connect it to AWS, verify a sender, send email through it from an app (REST API or the `seskit` Python SDK), receive mail, set up webhooks, upgrade a running instance, or fix one that is not sending (mail stuck at queued, unverified sender, forgotten password).
license: MIT
---

# SESKit

SESKit is a self-hosted email platform on Amazon SES. You run it on your own
server with your own AWS account, and get a REST API (`/v1`), a dashboard and a
Python SDK. Open source, MIT: <https://github.com/Otitodev/seskit>.
Docs: <https://otitodev.github.io/seskit/>.

It is two processes from one codebase, plus PostgreSQL and Redis:

- the **API** serves `/v1` and the dashboard
- the **worker** sends the mail, delivers webhooks and polls for delivery events

They never talk to each other. The API records a message and queues a job; the
worker sends it. So a send answers `queued`, not `sent`, and **if the worker is
not running, mail stays queued with no error**.

## Step 1: which situation is this?

Work this out before doing anything. Ask the user if you cannot tell.

| They have | Go to |
|---|---|
| No SESKit yet, or want to try it | **From scratch** below |
| A running instance and want to send from an app | [references/sending.md](references/sending.md) |
| A running instance that cannot send real mail yet | [references/aws.md](references/aws.md) |
| A running instance to upgrade, back up or fix | [references/operating.md](references/operating.md) |

To check whether an instance is up and healthy:

```bash
curl -fsS https://<their-url>/readyz     # 200 means the API can reach Postgres and Redis
```

`/healthz` only says the process is alive. Use `/readyz`.

## Rules that always apply

These come from how SESKit is built. Breaking one causes a problem that is
hard to see afterwards.

1. **Never put AWS credentials in `.env` or any environment variable.** SESKit
   takes an AWS access key through its own dashboard, per project, verifies it
   and stores it encrypted. There is no AWS setting on the server.
2. **Never change `SECRET_KEY` on an instance that has one.** The stored AWS
   keys are encrypted under a key derived from it. Changing it disconnects
   every project. If a `.env` already exists, leave it alone.
3. **Do not set `ALLOW_SIGNUP=true`.** The first registration claims the
   instance and signup then closes. That is the intended behaviour.
4. **Never expose PostgreSQL or Redis to the internet.**
5. **Never use `AdministratorAccess` or root credentials for AWS.** The exact
   policy is in [references/aws.md](references/aws.md).
6. **Do not print or log secrets**: `SECRET_KEY`, AWS keys, API keys (`sk_...`),
   webhook signing secrets, the recovery code.
7. **Do not install Docker, change a firewall or edit DNS** without asking.

## What you do and what the user does

Some steps belong to the user. Do the rest, then stop and tell them clearly.

| You can do | The user must do |
|---|---|
| Check Docker, clone, write `.env`, bring the stack up, run the doctor | Create the owner account and **save the recovery code** (shown once) |
| Write the IAM policy JSON for them to paste | Create the IAM user and access key in AWS, and paste it into the dashboard |
| Show the DNS records to add | Add them at their DNS host |
| Write the sending code in their app | Create the API key in the dashboard (shown once) and give it to the app as a secret |
| Suggest a TLS setup | Put TLS in front before real mail |

## From scratch

Ask two things first: **just trying it, or a real server?** and **do they have
an AWS account?** Neither blocks the first step.

**Trying it, no AWS account.** Mail is captured by Mailpit, so the whole
pipeline can be shown working in about a minute.

```bash
git clone https://github.com/Otitodev/seskit.git && cd seskit
cp .env.example .env
docker compose up -d --wait
```

Dashboard <http://localhost:8000>, API docs <http://localhost:8000/docs>,
Mailpit inbox <http://localhost:8025>.

**A real server.** Use the installer. It installs the latest release, writes a
`.env` with a generated `SECRET_KEY`, and is safe to run twice:

```bash
curl -fsSL https://otitodev.github.io/seskit/install.sh | sh
```

Then, in both cases:

1. Check it: `docker compose exec api python -m seskit_api.doctor`. Every check
   should pass. Each failure names the one thing to change.
2. The user opens the dashboard and creates the owner account, and saves the
   recovery code.
3. The user creates an API key, then you send a first email (see
   [references/sending.md](references/sending.md)).
4. For real mail: [references/aws.md](references/aws.md).
5. Before real mail on a server: put TLS in front and set `PUBLIC_BASE_URL`
   (see [references/install.md](references/install.md)).

Full detail, including the non-Docker path and what each service is:
[references/install.md](references/install.md).

## If something fails

Run the doctor first. Then see
[references/operating.md](references/operating.md#troubleshooting). The most
common causes, in order:

1. Mail stuck at `queued`: the worker is not running (`docker compose ps`).
2. `domain_not_verified`: the `from` address is not on a verified identity.
3. Rejected recipient: the AWS account is still in the SES sandbox.

## Where the truth lives

These files are a summary. When they disagree with the docs, the docs win:
<https://otitodev.github.io/seskit/>. A machine-readable index of every page is
at <https://otitodev.github.io/seskit/llms.txt>.
