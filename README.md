# google-calendar-sync

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build two-way Google Calendar sync in a
TypeScript backend. Hosts connect their Google account. Events on the calendars they choose block their
availability, kept current by push notifications and a renewal cron. Confirmed bookings are written to
their calendar as events, with Google Meet links when asked. Firestore and Supabase/Postgres are both
carried.

Busy time is a disposable mirror of a time window, rebuilt by a full resync and diffed by deterministic
ids. Booking events are addressed by an id derived from the booking, and never read back as busy time.
Most failures of calendar integrations come from one direction leaking into the other, and the skill
is built to keep them apart.

It was extracted from the calendar integration of a production booking API and audited before
extraction, together with the slot engine that consumes its output. Eleven defects it found are fixed
in the templates, most of them pinned by a test. Among them: a booking's own event blocking the rest of
a group session, renewals that silently left calendars without a channel, phantom busy blocks from
concurrent notifications, multi-day vacations blocking only their first day, and calendars truncated
at 2500 events. [`references/provenance.md`](references/provenance.md) records what changed, what was
kept deliberately and why.

## Install

```bash
npx skills add <owner>/google-calendar-sync
```

Or copy this folder into your agent's skills directory (`~/.claude/skills/`, `~/.codex/skills/`,
`~/.gemini/skills/`).

## Contents

| File | What it answers |
|---|---|
| [`SKILL.md`](SKILL.md) | When to use it, critical facts, hard rules, the seam contract |
| [`references/data-model.md`](references/data-model.md) | Records, types, the store contract, the rename table |
| [`references/oauth-and-tokens.md`](references/oauth-and-tokens.md) | Google Cloud setup, scopes, the OAuth round trip, token lifetime, encryption |
| [`references/google-api.md`](references/google-api.md) | The fetch-based client and the API facts that shape it |
| [`references/busy-sync.md`](references/busy-sync.md) | Which events block, the sync run, feeding availability |
| [`references/booking-events.md`](references/booking-events.md) | Writing and deleting booking events, Meet links |
| [`references/channels-and-settings.md`](references/channels-and-settings.md) | Watch channels, renewal, settings transitions, the calendar list |
| [`references/http-routes.md`](references/http-routes.md) | Route handlers, the webhook, Next.js and Express wiring |
| [`references/firestore-backend.md`](references/firestore-backend.md) | Collections, index, rules, store |
| [`references/supabase-backend.md`](references/supabase-backend.md) | Schema, RLS, optimistic concurrency, store |
| [`references/operations.md`](references/operations.md) | Health, logs, configuration, runbook |
| [`references/testing.md`](references/testing.md) | Running the suite, what it pins, mutation check, doubles |
| [`references/test-suite.md`](references/test-suite.md) | The 32 tests |
| [`references/provenance.md`](references/provenance.md) | Defects fixed, choices kept, additions |

## License

MIT
