# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.1] - 2026-10-16

### Changed

- Outlook / Microsoft 365 now points to the sibling skill `microsoft-calendar-sync`.

## [0.1.0] - 2026-10-16

First release, extracted from a production booking API's Google Calendar integration and hardened.

### Added

- OAuth connection with single-use server-side state, PKCE, granted-scope check, AES-256-GCM token
  storage, refresh with `invalid_grant` → `reauth_required`, same-account reconnect that keeps settings,
  and disconnect with revocation.
- A dependency-free Google Calendar client on `fetch`.
- Busy sync: event rules (free, declined, own bookings, working location, all-day spans across DST),
  every page of the window, a row-based diff over deterministic ids, and a single-flight lease with a
  queued rerun.
- Booking events with ids derived from the booking, 409 → update, private marker, Meet links, backfill.
- Watch channels with signed channel-id tokens, make-before-break renewal, periodic resync, orphan
  clean-up, and settings transitions.
- Web `Request`/`Response` route handlers with Next.js and Express wiring.
- Firestore and Supabase/Postgres stores, the Postgres schema with RLS.
- Derived per-calendar health and an operations runbook.
- A 32-test `node:test` suite with a memory store and a fake Google, run under four process time zones.
