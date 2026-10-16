# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-10-16

The skill brought to the Timerise skill standard, with a new adaptation reference, agent evals, and one
template fix found by compiling every block on its own.

### Added

- `references/adaptation.md`: host prerequisites, the seam contract, the order of work and the
  non-negotiables, linked from the quick start and the reference directory.
- `evals/prompts.md` with three operator prompts, and the agent eval workflow that runs prompt 1 on every
  published release.
- README sections in the shared order: install with `npx skills add`, activation, the five
  non-negotiables with what verifies each, requirements, verification, neighbouring skills, contributing.

### Changed

- The `SKILL.md` description follows the standard order and names `time-slot-generator`; the hard rules
  are numbered and match the README and `adaptation.md`.
- The origin is stated as an audit against the earlier implementation, in the README, `SKILL.md`,
  `provenance.md` and this file.
- The rename table lists host words only.
- Prose, tables, comments and diagrams use plain ASCII punctuation. Code changed only in comments.

### Fixed

- `app/api/calendar/google/calendars/[calendarId]/route.ts` has its own code block and imports
  `calendarRoutes`; it shared a block with the `[action]` route and did not compile when copied alone.

## [0.1.1] - 2026-10-16

### Changed

- Outlook and Microsoft 365 now point to the sibling skill `microsoft-calendar-sync`.

## [0.1.0] - 2026-10-16

First release, written by the engineer who owns the calendar integration of a booking API and audited
against the earlier implementation of that integration.

### Added

- OAuth connection with single-use server-side state, PKCE, granted-scope check, AES-256-GCM token
  storage, refresh that turns `invalid_grant` into `reauth_required`, same-account reconnect that keeps
  settings, and disconnect with revocation.
- A dependency-free Google Calendar client on `fetch`.
- Busy sync: event rules (free, declined, own bookings, working location, all-day spans across DST),
  every page of the window, a row-based diff over deterministic ids, and a single-flight lease with a
  queued rerun.
- Booking events with ids derived from the booking, 409 turned into an update, private marker, Meet links,
  backfill.
- Watch channels with signed channel-id tokens, make-before-break renewal, periodic resync, orphan
  clean-up, and settings transitions.
- Web `Request`/`Response` route handlers with Next.js and Express wiring.
- Firestore and Supabase/Postgres stores, the Postgres schema with RLS.
- Derived per-calendar health and an operations runbook.
- A 32-test `node:test` suite with a memory store and a fake Google, run under four process time zones.
