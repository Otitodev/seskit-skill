# SESKit skill

An agent skill that teaches a coding agent how to set up, connect, use and
upgrade [SESKit](https://github.com/Otitodev/seskit), the self-hosted email
platform built on Amazon SES.

It covers both starting points:

- **From scratch:** install SESKit, connect AWS, verify a sender, send a first email.
- **A running instance:** send from an app, set up webhooks, upgrade, back up, and fix what is not working.

It also carries the rules that are easy to break by accident: AWS keys go in the
dashboard and never in `.env`, `SECRET_KEY` is never changed, and the steps that
belong to the user (owner account, AWS key, DNS, TLS) are left to them.

## Install

For Claude Code:

```bash
git clone https://github.com/Otitodev/seskit-skill.git
cp -r seskit-skill/skills/seskit ~/.claude/skills/
```

Or with the [`skills`](https://github.com/vercel-labs/skills) installer:

```bash
npx skills add Otitodev/seskit-skill
```

An agent with no skills system can read the file directly:
<https://raw.githubusercontent.com/Otitodev/seskit-skill/main/skills/seskit/SKILL.md>

## It is a summary

The skill is distilled from the [SESKit docs](https://otitodev.github.io/seskit/)
and links back to them. If they disagree, the docs are right.

## License

MIT, as SESKit.
