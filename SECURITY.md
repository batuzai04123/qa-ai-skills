# Security Policy

## Reporting a vulnerability

Please **do not open a public issue** for security problems.

Report them privately through GitHub:
**[Report a vulnerability](https://github.com/batuzai04123/qa-ai-skills/security/advisories/new)**
(Security tab → *Report a vulnerability*).

Please include:
- which skill and section is affected
- what an agent following it could be led to do (for example: leak credentials into a report,
  write to a shared system without approval, run mutating steps against a read-only environment)
- steps to reproduce, with any sensitive data removed

You can expect an acknowledgement within 7 days. Once a fix is merged, the report is credited
in the skill's `CHANGELOG.md` unless you ask otherwise.

## Scope

In scope:
- Skill instructions that could cause an agent to expose secrets or personal data
- Instructions that bypass the human-review or read-only safeguards
- Reference code in this repository (capture helpers, renderers, upload snippets)

Out of scope:
- Vulnerabilities in third-party tools the skills use (Playwright, Appium, Google APIs, AI agents) —
  report those to their maintainers
- The behaviour of the application you are testing

## If you committed a secret

Rotate the secret first, then contact the maintainer. Removing the commit is not enough once it
has been pushed.
