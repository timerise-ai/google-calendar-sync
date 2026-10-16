# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json`, no build and
nothing here executes. It teaches an agent to build two-way Google Calendar sync in *someone else's*
TypeScript backend, on Firestore or Supabase/Postgres.

The code blocks in `references/` are the templates an agent copies. They are kept as complete files (one
fenced block per file, headed by a `// calendar-sync/<file>.ts` comment, or `// wiring/…`, `// server/…`
and `// app/…` for the wiring examples) so they can be extracted verbatim and checked. Do not add a
`package.json` or test config to make them runnable here.

## Structure

- `SKILL.md`: the router, loaded on every activation. Keep it at or under 150 lines. The `description`
  frontmatter is the activation trigger. The body holds when/when-not, the architecture, seven critical
  facts, five hard rules, the Adaptation Contract and the reference directory.
- `references/`: one question per file. `provenance.md` is the rationale layer; read it before
  "simplifying" anything, because most odd-looking choices are listed under "Kept deliberately".

## Verifying a change

Template edits must be re-verified the way 0.1.0 was:

1. Extract every `// calendar-sync/*.ts` block, the wiring blocks and `schema.sql` into a scratch
   project, at the paths in their header comments (`wiring/next-routes.ts` imports `@/calendar-sync/setup`).
2. `tsc --strict` passes, and also with `--noUncheckedIndexedAccess`. Borrow `firebase-admin`,
   `@supabase/supabase-js`, `express`, `@types/express` and `@types/node` from any project.
3. `node --test` passes under `TZ=UTC`, `Europe/Warsaw`, `Pacific/Auckland` and `America/Los_Angeles`.
   Rewrite the extensionless imports to `.ts` for Node's type stripping.
4. Every mutation in the table in `references/testing.md` still turns the suite red.
5. `schema.sql` applies to a scratch Postgres.

Never change `bookingEventId`, `busyBlockId` or `calendarRowId` inputs. Every consumer's existing events,
blocks and rows are addressed by them.

## Releases

Bump `CHANGELOG.md` with Keep a Changelog headings. Patch for template fixes, minor for new capability.
