# google-calendar-sync

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build **two-way Google Calendar sync** in a
**Next.js App Router** app, with an Express adapter for other Node servers. Hosts connect their Google
account. Events on the calendars they choose block their availability, kept current by push notifications
and a renewal cron. Confirmed bookings are written to their calendar as events, with Google Meet links when
asked. Firestore and Supabase/Postgres are both carried.

The design turns on keeping the two directions apart. **Busy time is a disposable mirror of a time window,
rebuilt by a full resync and diffed by deterministic ids. Booking events are addressed by an id derived from
the booking, and never read back as busy time.**

This skill was written by the engineer who owns the calendar integration of a booking API, and audited
against the earlier implementation of that integration together with the slot engine that consumes its
output. The templates hold the properties a calendar sync has to hold: a booking's own event never becomes
busy time; a burst of notifications converges to one block per event; every page of a calendar is read,
and a failed page deletes nothing; a multi-day all-day event blocks every day, whatever the server's time
zone; a channel is renewed before the old one is stopped; a booking pushed twice is one event; revoked
access becomes a reconnect prompt that keeps every setting. 32 tests state them, run under four process
time zones, and [`references/provenance.md`](references/provenance.md) has the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/google-calendar-sync
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/google-calendar-sync -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/google-calendar-sync.git ~/.claude/skills/google-calendar-sync
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/google-calendar-sync ~/.agents/skills/google-calendar-sync
```

Update with `git pull` in the clone. The current release is **0.2.2**. See [CHANGELOG.md](CHANGELOG.md).

## Activation

The skill activates on tasks such as "let staff connect their Google Calendar", "block availability when
the host is busy in Google", "put bookings into the host's calendar with a Meet link", or "our calendar sync
shows phantom busy time". Each host matches skills its own way, so on a first run invoke it explicitly:
`/google-calendar-sync` in Claude Code, `$google-calendar-sync` in Codex CLI, and through `/skills` in
Gemini CLI.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: trigger, architecture, adaptation contract, critical facts, hard rules, quick start, reference directory |
| `references/adaptation.md` | Host prerequisites, seam contract, order of work, the non-negotiables |
| `references/data-model.md` | Records, types, the store contract, the rename table |
| `references/oauth-and-tokens.md` | Google Cloud setup, scopes, the OAuth round trip, token lifetime, encryption |
| `references/google-api.md` | The fetch-based client and the API facts that shape it |
| `references/busy-sync.md` | Which events block, the sync run, feeding availability |
| `references/booking-events.md` | Writing and deleting booking events, Meet links |
| `references/channels-and-settings.md` | Watch channels, renewal, settings transitions, the calendar list |
| `references/http-routes.md` | Route handlers, the webhook, Next.js and Express wiring |
| `references/firestore-backend.md` | Collections, index, rules, store |
| `references/supabase-backend.md` | Schema, RLS, optimistic concurrency, store |
| `references/operations.md` | Health, logs, configuration, runbook |
| `references/testing.md` | Running the suite, what it pins, mutation check, doubles |
| `references/test-suite.md` | The 32 tests |
| `references/provenance.md` | The audit ledger: fixed, kept deliberately, changed, designed here |
| `evals/` | `prompts.md` with what an operator types, and one file per agent eval run |
| `.github/workflows/agent-eval.yml` | Runs the agent evals on every published release |

The seam is the `CalendarStore` and `BookingSource` interfaces plus `authenticate(request)`. They bound
everything the host app supplies: its database, its auth, its booking status flow. Availability is the
host's slot engine reading the busy blocks, and the settings screen is built with the host's components on
the JSON the routes return.

## The five non-negotiables

1. **Never take the user id from the OAuth callback URL.** The state is a key to a server-side, single-use,
   ten-minute record; otherwise anyone attaches their calendar to someone else's account. The connect tests
   consume a state twice, let one expire, and check the user comes from the server.
2. **Never trust a notification before its channel token verifies against the channel id it arrived on.**
   The endpoint is public; the token is the only credential. The webhook test sends a forged token, a
   foreign resource and an unknown channel.
3. **Never stop the old channel before the new one exists.** If the new watch fails, the calendar has no
   channel and its busy time stops updating while the UI says "on". The renewal tests fail the new watch
   and expect the old channel kept, and remove a channel and expect the cron to open one.
4. **Never store tokens in clear text.** A refresh token is long-lived read and write access to a calendar.
   The token test round-trips AES-256-GCM and rejects tampering; the connect test reads the stored tokens
   back encrypted.
5. **Never let a Google failure fail a booking.** Outbound writes log and return; the booking stands.
   `pushBookingEvent` and `removeBookingEvent` catch every Google error and return a result per calendar,
   as [booking-events.md](references/booking-events.md) shows.

Everything else, the names, the settings screen, the database, the scheduler, is the host app's.

## Requirements

Node.js on the server, TypeScript, and `firebase-admin` or `@supabase/supabase-js`; no Google SDK. A Google
Cloud project with an OAuth client, a public HTTPS URL for Google's notifications, and a scheduler that
calls one URL every 12 hours. Google requires verification of the calendar scopes before the public can
connect. Running the suite directly needs Node 22.18 or later, which strips types; any runner with a
TypeScript transform runs it unchanged.

## Verification

Every block is written to the path in its header comment and compiled as one project with `tsc` 6.0.3 under
`strict` and `noUncheckedIndexedAccess`, with `firebase-admin` 13.6.1, `@supabase/supabase-js` 2.111.0,
`express` 5.2.1 and `@types/node`. The 32 tests pass under `node --test` with `TZ` set to `UTC`,
`Europe/Warsaw`, `Pacific/Auckland` and `America/Los_Angeles`, and each mutation in the table in
[testing.md](references/testing.md) turns the suite red. `schema.sql` applies to a fresh PostgreSQL 18 with
RLS on every table. `next build` 16.4.2 of the two route files succeeds with no environment variable set.

## Not this

| Problem | Go to |
|---|---|
| Outlook and Microsoft 365 calendars | `microsoft-calendar-sync` |
| Generating bookable slots and subtracting busy time | `time-slot-generator` |
| Public booking pages with Google and Outlook sync built in | `calendar-scheduling-links` |
| A calendar UI or invitation e-mails | The host's own UI and e-mail |
| Availability from a whole Workspace without per-user OAuth | Google's free/busy query with domain-wide delegation, not templated here |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every block names its destination on the first line, every TypeScript block is written to compile as one
project under `strict` and `noUncheckedIndexedAccess`, and the suite passes under `node --test` in four
process time zones. Claims in this skill are meant to be verifiable: if you change a factual claim, say how
you verified it, whether against the Google Calendar API reference, Google's OAuth 2.0 documentation, the
Firebase Admin or Supabase documentation, the IANA time zone database, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
