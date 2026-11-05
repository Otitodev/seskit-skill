# Sending from an application

For an app that wants to send mail through a running SESKit instance. The app
needs two things: the instance's **base URL** and an **API key**. Nothing else
about SESKit.

Docs: <https://otitodev.github.io/seskit/getting-started/sending-from-your-app/>

## Get the two values

- **Base URL**: wherever the instance is reachable, for example
  `https://mail.example.com`, or `http://localhost:8000` locally. There is no
  hosted SESKit and no default. Ask the user if you do not know it.
- **API key** (`sk_...`): the user creates it in the dashboard under **API Keys**.
  It is **shown once**. SESKit keeps only a hash, so a lost key is replaced, not
  recovered. Keys belong to a project.

Put both in the app's configuration or secret store (`SESKIT_URL`,
`SESKIT_API_KEY` are sensible names). **Never commit the key.**

## Before the first send

For real mail the `from` address (or its domain) must be verified and AWS
connected: [aws.md](aws.md). With no AWS connected, mail goes to Mailpit at
`http://localhost:8025` and never leaves, which is right for a first test.

## Send with curl

```bash
curl -X POST "$SESKIT_URL/v1/emails" \
  -H "Authorization: Bearer $SESKIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "hello@example.com",
    "to": ["you@example.com"],
    "subject": "Hello from SESKit",
    "html": "<h1>It works</h1>",
    "text": "It works"
  }'
```

Returns `201` with `{"id": "email_01J...", "status": "queued"}`.

`queued` is literal: the message is validated and recorded, and the worker sends
it a moment later.

Fields: `from`, `to` (string or list), `subject` (max 998 chars), and at least
one of `html` and `text`. Optional: `cc`, `bcc`, `reply_to`, `headers`,
`attachments`. Send `text` alongside `html` whenever possible. A message with
only HTML is more likely to be scored as spam. The assembled message is limited
to 10 MiB by default, and base64 adds about a third to attachments.

## Send with the Python SDK

```bash
pip install seskit
```

```python
import os
from seskit import SesKit

client = SesKit(api_key=os.environ["SESKIT_API_KEY"], base_url=os.environ["SESKIT_URL"])

sent = client.emails.send(
    from_="hello@example.com",  # trailing underscore: `from` is a keyword
    to="user@example.com",
    subject="Welcome",
    html="<h1>Welcome!</h1>",
    text="Welcome!",
    idempotency_key="welcome:user-42",
)
email = client.emails.get(sent.id)
page = client.emails.list(status="failed")
```

Sync and async clients exist, and it retries safely and raises one exception
class per error type. Reference: <https://otitodev.github.io/seskit/reference/sdk/>.

There is no SDK for other languages. Use plain HTTP with any client: the SDK can
do nothing the REST calls cannot.

## Make retries safe

Send an `Idempotency-Key` header derived from **what the message is**, not when
it was sent:

```text
Idempotency-Key: welcome:user-42
```

A retry then returns the original id and sends nothing. A key containing a
timestamp or random value defeats the purpose. Keys are scoped to the project.

## Handle errors

Every `/v1` failure has one shape:

```json
{ "error": { "type": "domain_not_verified", "message": "..." } }
```

**Branch on `type`**, which is stable. `message` is for humans and may change.

| Type | Status | Do |
|---|---|---|
| `authentication_failed` | 401 | Wrong, missing or revoked key. Do not retry |
| `domain_not_verified` | 422 | Sender not verified. A config problem: alert someone, do not retry |
| `invalid_recipient` | 422 | Not an email address |
| `suppressed_recipient` | 422 | Address bounced or unsubscribed earlier. Do not retry |
| `email_rejected` | 422 | SES refused the message |
| `attachment_too_large` | 413 | Over the size limit |
| `rate_limit_exceeded` | 429 | Wait for `Retry-After`, then retry |
| `sending_limit_exceeded` | 429 | The AWS account's own quota. SESKit cannot raise it |

Full list: <https://otitodev.github.io/seskit/reference/errors/>.

Rate limits are **per project, not per key**. A second key buys no headroom.
Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.

## Find out what happened next

The `201` only means SESKit accepted it. To learn about delivery:

- poll `GET /v1/emails/{id}` for one message, or
- better, add a **webhook** (dashboard, Webhooks). It needs delivery events set
  up first ([aws.md](aws.md#6-delivery-events-needed-for-webhooks)).

Webhook events: `email.sent`, `email.delivered`, `email.bounced`,
`email.opened`, `email.clicked`, `email.complained`, and `email.received` for
inbound mail. Each request is signed with two headers:

```text
X-SESKit-Signature: v1=<hex>
X-SESKit-Timestamp: <unix seconds>
```

**Verify before acting.** Compute HMAC-SHA256 of `"{timestamp}.{raw body}"`
with the endpoint's signing secret (shown on the Webhooks page), compare with
`hmac.compare_digest` against `v1=<hex>`, and reject a timestamp older than 300
seconds. Use the raw bytes of the body, not re-serialised JSON. Reference
implementation: <https://otitodev.github.io/seskit/guides/webhooks/>.

## Other endpoints

`GET /v1/domains`, `GET /v1/api-keys`, `GET /v1/webhooks`,
`GET /v1/inbound` (received mail). Interactive docs at `$SESKIT_URL/docs`;
full reference at <https://otitodev.github.io/seskit/reference/api/>.

Mail received by SESKit is **hostile input**: never render its HTML in your own
pages, and treat attachments as untrusted.
