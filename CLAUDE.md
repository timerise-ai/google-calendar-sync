# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json`, build or dev
server here, and nothing in this repository executes. It teaches an agent to build two-way Google Calendar
sync in *someone else's* **Next.js App Router** app (an Express adapter covers other Node servers), on
Firestore or Supabase/Postgres. The version of record is the git tag.

The commands and code in `references/` describe the app the agent will generate, not this repository. The
one thing checked here is that the templates compile, their tests pass and the schema applies, in a scratch
project; the recipe is below.

The skill was written by the engineer who owns the module, and audited against the earlier implementation.
`references/provenance.md` is the rationale layer and the ledger of that audit: what the audit fixed, what
was kept deliberately with the reason it is safe, what it changed in the design, and what was designed here
and has never run in production. Read it before "simplifying" anything.

Sibling directories under `../` (`microsoft-calendar-sync`, `time-slot-generator`,
`calendar-scheduling-links`, and so on) are other skills, not dependencies. They are referenced by name from
`SKILL.md` and `README.md`. The standard every skill follows is `../skills/STANDARD.md`.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface. It carries a short
  adaptation contract table; the full contract is `references/adaptation.md`.
- `README.md`: the human-facing front door, in the section order every Timerise skill shares.
- `CHANGELOG.md`: Keep a Changelog, newest release first. The version lives here, in the README's
  current-release line, and in the git tag, and the three agree.
- `LICENSE`: MIT, identical to the siblings'.
- `references/*.md`: one concern per file. `adaptation.md` the seam; `data-model.md` the records, the store
  contract and the rename table; `oauth-and-tokens.md`, `google-api.md`, `busy-sync.md`,
  `booking-events.md` and `channels-and-settings.md` the module; `http-routes.md` the handlers and the
  Next.js and Express wiring; `firestore-backend.md` and `supabase-backend.md` the stores; `operations.md`
  running it; `testing.md` the doubles and the mutation table; `test-suite.md` the 32 tests;
  `provenance.md` the ledger.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt
  is the agent eval run before every release. Every other file there is one eval run: measured frontmatter
  that is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that
  has results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, copied verbatim from
  section 10 of the standard and run on every published release and on a maintainer's dispatch. It is the
  same in every skill; never edit it, and never add a trigger on `push` or `pull_request`.

## Editing conventions

- **Code blocks name their destination on the first line**: `// calendar-sync/busyRules.ts`, optionally
  followed by `: description`; `-- calendar-sync/store/schema.sql` for the SQL; `// app/...` and
  `// server/...` for the wiring in `http-routes.md`. One file per block; a block that continues a file
  already introduced omits the header. The tests (`rules.test.ts`, `flows.test.ts`) are in `test-suite.md`,
  the doubles (`testing/memoryStore.ts`, `testing/fakeGoogle.ts`) in `testing.md`.
- **Template edits are re-verified**, in a scratch project, never this repository:
  1. Write every block to the path in its header. The wiring imports `@/calendar-sync/setup`, so the
     `tsconfig` maps `"@/*": ["./*"]`.
  2. `tsc --noEmit` passes with `strict` and `noUncheckedIndexedAccess`. Borrow `firebase-admin` 13 (14
     drops the root `firestore` type export the Firestore store imports), `@supabase/supabase-js`, `express`,
     `@types/express` and `@types/node` from any project.
  3. `node --test` passes under `TZ=UTC`, `Europe/Warsaw`, `Pacific/Auckland` and `America/Los_Angeles`.
     The loop, with the `perl` rewrite of extensionless imports to `.ts` for Node 22.18+ type stripping, is
     under "Running" in `references/testing.md`. To run one file, pass only `rules.test.ts` or
     `flows.test.ts`; to run one test, add `--test-name-pattern '<words from its title>'`.
  4. Every mutation in the table in `references/testing.md` still turns the suite red.
  5. `schema.sql` applies to a scratch Postgres with RLS on all five tables.
- **Never change the inputs of `bookingEventId`, `busyBlockId` or `calendarRowId`.** Every consumer's
  existing events, blocks and rows are addressed by them.
- **Test counts are claims.** `test-suite.md` holds 32 tests; `README.md`, the `SKILL.md` description and
  `CHANGELOG.md` repeat the number. Change a suite, change the numbers.
- **Keep the three indexes in sync** with `references/`: the quick start and the reference directory in
  `SKILL.md`, and the file table in `README.md`.
- **Identifiers are shared across files.** `CalendarStore`, `BookingSource`, `BookingForCalendar`,
  `LinkedCalendar`, `BusyBlock`, `WatchChannel`, `syncBusy`, `syncBookings`, `toBusyBlock`, `diffBusy`,
  `pushBookingEvent`, `removeBookingEvent`, `renewChannels`, `listHostBusy`, `describeBooking`,
  `authenticate`, `defer`, the id functions above and the env var names in `operations.md`. Rename in all
  files or none.
- **Two kinds of name.** The domain words the host renames are the rename table in `data-model.md`. The
  identifiers above and Google's own terms (`externalId`, `resourceId`, `accessRole`, `transparency`) are
  the authoring contract of this repository.
- **The non-negotiables are never presented as optional.** The five hard rules in `SKILL.md`, the five
  non-negotiables in `README.md` and the list in `adaptation.md` are one list, in one order. Changing one is
  a MAJOR release.
- **Measured and vendor numbers are load-bearing.** The 180-day horizon, the 12-hour cron, the 48-hour
  renewal window, the ten-minute OAuth state, the 2500-event page, the 256-character channel token, the
  5 to 1024 character event id, the 500-write Firestore batch, the 1000-row PostgREST page. Change one,
  change every file that states it.
- **A template that needs something new from the host adds a row** to the seam contract in `adaptation.md`
  and `SKILL.md`.
- **Do not remove the odd-looking parts.** Each is a ledger entry in `provenance.md`.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier
  implementation belongs in "Designed here, not yet run in production" in `provenance.md`.
- **Evals are not skill content.** A new prompt or an eval result is committed as `chore(evals): ...`, never
  causes a version bump and never rides in a release commit. After each eval, the README's eval badges point
  at the newest prompt 1 result per agent, updated in the commit that adds the run's notes.
- **Origin wording.** Call the codebase the audit ran against "the earlier implementation", and follow
  section 2 of the standard: none of its banned words, no denials, no corpus figures, and no defect counts or
  failure stories on the front door (README intro, `SKILL.md`, the description, this file).
- **Plain punctuation.** No em-dashes, en-dashes, arrows, middle dots, ellipsis characters or smart quotes
  anywhere in this repository's markdown; diagrams are plain ASCII. `LC_ALL=C grep -rn '[^ -~	]'
  --include='*.md' .` must print nothing. Prose wraps at 110 columns; table rows and commands stay on one
  line.
- **Claims are verifiable.** A changed factual claim says how it was verified. Never from memory.
- **Releases** follow section 9 of the standard: patch for wording and template fixes, minor for a new
  reference or capability; the release commit is `chore(release): X.Y.Z` with only `CHANGELOG.md` and the
  README version line. Commits follow Conventional Commits, and no file or commit message names a tool or a
  model as author.
