# Agent skills for client portals

Skills here are meant to be copied **into a client's portal repository**, not
used in this repo. They teach an AI coding assistant how to integrate with this
payment service correctly — in particular the parts that are easy to get subtly,
expensively wrong (signature verification, idempotency, `expired` not being
final).

## pay-integration

Everything a portal needs to take a payment: create order → hosted checkout →
verify signed return/webhook → poll `GET /orders/{id}` → fulfil idempotently →
sweep.

### Install

**Claude Code** — install as a plugin:

```
/plugin marketplace add kerala-startup-mission/pay-skill
/plugin install pay@pay-skill
```

The assistant picks it up automatically when a task mentions payments,
checkout, orders, or the Pay API. Force it with `/pay:pay-integration`.

Manual alternative: copy `skills/pay-integration` into the portal repo's
`.claude/skills/` (or `~/.claude/skills/` for all repos).

**OpenAI Codex** — ask the built-in installer:

```
$skill-installer install https://github.com/kerala-startup-mission/pay-skill/tree/main/skills/pay-integration
```

Manual alternative: copy `skills/pay-integration` into the portal repo's
`.agents/skills/` (or `~/.agents/skills/` for all repos). Invoke explicitly
with `$pay-integration`.

**Cursor / Windsurf / Copilot / other assistants** — copy
`skills/pay-integration` to `docs/pay-integration/` in the portal repo and
point the tool's rules file at it, e.g. in `.cursor/rules/pay-integration.mdc`
or `AGENTS.md`:

```
When implementing anything involving payments, checkout, orders or the Pay API,
read docs/pay-integration/SKILL.md and follow it exactly.
```

### Contents

| file | what |
|---|---|
| `SKILL.md` | the procedure, the rules, the anti-patterns, the checklist |
| `agents/openai.yaml` | Codex display metadata (ignored by Claude Code) |
| `references/api-reference.md` | endpoints, fields, statuses, errors, rate limits |
| `references/php-laravel.md` | Laravel client, verifier, routes, sweep, tests |
| `references/node-express.md` | Express/Next client, verifier, routes, sweep, tests |

`SKILL.md` is what the assistant loads first; it pulls in a reference only when
the task needs it.

### Keeping it in sync

The skill restates the contract in [`../integration.md`](../integration.md).
When the API contract changes — canonical signing string, field rules, statuses,
error codes — update **both**, or portals will be built against a stale spec.
