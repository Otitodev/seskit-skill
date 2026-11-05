# Connecting SESKit to AWS

Until a project connects AWS, it sends through Mailpit. Once connected and the
`from` address is verified, it sends through SES. **Nothing changes in the
calling code**: same endpoint, same request.

A project that has AWS connected but an unverified sender gets an error, not a
quiet local delivery. That is intentional.

Docs: <https://otitodev.github.io/seskit/guides/connect-aws/>

## The credential rule

The access key goes into the **dashboard AWS page**, never into `.env`, never
into chat history, never onto the server by hand. The user does this part.
SESKit checks the key against AWS before storing it, then keeps it encrypted
under a key derived from `SECRET_KEY`.

No AWS account yet: <https://otitodev.github.io/seskit/guides/no-aws-account-yet/>.
Needs a card and a phone number. Pick a region and **stay in it**. SES
identities, queues and configuration sets are per region.

## 1. The IAM policy

Never `AdministratorAccess`, never root credentials. Give the user this to paste
at **IAM, Policies, Create policy, JSON**, and name it `seskit-sending`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity",
        "ses:GetAccount",
        "ses:CreateEmailIdentity",
        "ses:GetEmailIdentity",
        "ses:DeleteEmailIdentity",
        "ses:SendEmail",
        "ses:PutAccountDetails"
      ],
      "Resource": "*"
    }
  ]
}
```

Only the first two are needed to connect and look around; connecting is
read-only and creates nothing. Delivery events and receiving mail need more
actions, listed in <https://otitodev.github.io/seskit/guides/iam-policies/>.
**Fetch that page for the current list rather than guessing.**

## 2. The user and key

1. **IAM, Users, Create user**, named `seskit`. Attach `seskit-sending`
   directly. Leave console access **off**.
2. Open the user, **Security credentials, Create access key**, choose
   **Application running outside AWS**.
3. The secret is shown **once**. The user copies both values.

## 3. Paste it into SESKit

Dashboard, **AWS**, choose the region the SES account is in, paste both values,
**Connect**. The page then shows the account id, region, sending quota and
whether the account is in the sandbox.

To rotate: make a second key on the same user, use **Replace the access key**,
delete the old one in IAM once mail flows. Do not press **Disconnect** to rotate:
it also tears down the event infrastructure.

## 4. Verify a sender

SES will not send from an unverified address. Dashboard, **Domains**.

| | Setup | Time | Sends as |
|---|---|---|---|
| Email address | Click a link SES emails | Minutes | That one address |
| Domain | Three CNAME (DKIM) records at the DNS host | Usually under an hour, up to 72h | Anything on the domain |

**Start with an address.** It needs no DNS, so it is the fastest route to a real
send. A value with `@` is treated as an address, anything else as a domain.

The DKIM records must stay in place permanently. SESKit rechecks verified
identities monthly and will notice if they disappear.

SPF and DMARC are the user's to publish. SESKit does not manage them. Start
DMARC at `p=none` and read the reports before tightening. A custom MAIL FROM
subdomain makes SPF align with the user's own domain.

## 5. The SES sandbox

Every new AWS account starts in the sandbox: 200 messages per 24 hours, 1 per
second, and **only to verified recipients**.

If a first real send fails with a rejected recipient, this is almost always why.
Meanwhile:

- verify their own addresses and send to those
- use `bounce@simulator.amazonses.com` and `complaint@simulator.amazonses.com`,
  which need no verification and are the safe way to test bounce handling
- keep using Mailpit for anything that need not leave

To leave: dashboard **AWS** page, request production access. It is a support
review, usually answered within 24 hours. SESKit first requires a verified
**domain**, delivery events set up, and one delivered test message. Start this
early because it is the long wait.

## 6. Delivery events (needed for webhooks)

Dashboard **AWS** page, set up delivery events. This **creates** an SQS queue,
an SNS topic and an SES configuration set in the user's account. It says what it
will create first. It needs the extra IAM actions above. Without it SES reports
nothing back, so there are no delivered/bounced statuses and no webhooks.

Disconnecting removes what it created.

## 7. Receiving mail (optional)

Sending needs none of this. Receiving creates an S3 bucket, edits SES receipt
rules, and needs its own third IAM policy and an MX record. Read
<https://otitodev.github.io/seskit/guides/receive-email/> before offering it.

## Cost

About $0.10 per thousand emails, plus a few cents for SNS and SQS traffic.
