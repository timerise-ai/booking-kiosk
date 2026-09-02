# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.4] - 2026-09-02

Wording release. Templates and technical content are unchanged from 0.1.3.

### Changed
- The front door, `README.md`, `SKILL.md` and `CLAUDE.md`, describes the module by the
  properties the reducer suite, the guard tables and the failure-mode table verify: prices,
  capacity and promo validity computed on the server, idempotent submits, a required device
  header, stock locks released on every failure path, an in-memory session that resets. The
  audit record stays in `references/provenance.md`, linked from each of them.

## [0.1.3] - 2026-09-02

Wording release. The origin and audit statements across the skill follow section 2 of
the skill standard; templates and technical content are unchanged from 0.1.2. The
repository history starts at this release.

### Changed
- Origin and audit wording across `SKILL.md`, `README.md`, `CLAUDE.md` and `references/` now
  follows the skill standard: the reference point is the earlier implementation, stated
  in the standard's own words. Technical content is unchanged.

## [0.1.2] - 2026-09-02

Documentation-only release. The skill itself, `SKILL.md` and `references/`, is
unchanged from 0.1.1.

### Changed
- README: the install leads with `npx skills add timerise-ai/booking-kiosk`, which installs the skill
  into every skills-compatible agent it detects, with the `-a` form for named agents; the
  Claude Code clone moves under a *Manual install* heading. Activation gets its own
  heading, and a *Not this* table points neighbouring problems to the right skill or tool.
- README: the skill's origin is reworded. It was written by the engineers who built the
  module it describes; the reference point for `provenance.md` is the earlier
  implementation rather than "the source"; the index is called Timerise Skills.
- README: every em-dash, arrow and en-dash in the prose is rewritten as a comma, colon,
  full stop or conjunction.

## [0.1.1] - 2026-09-01

### Added
- `README.md` describing the skill, its install commands for Claude Code, Codex
  CLI and Gemini CLI, the contents of each reference file, and the five
  non-negotiables — the hard rules from `SKILL.md`, plus the `SET_SLOT` /
  `REFRESH_SLOT` distinction that is the module's signature. The seam contract
  points at the Adaptation Contract table in `SKILL.md`, since this skill keeps
  it there rather than in a `references/adaptation.md`.
- MIT `LICENSE`, naming Timerise Sp. z o.o., matching every other skill in the
  [index](https://github.com/timerise-ai/skills).

### Changed
- `CLAUDE.md` records the new packaging: the tree lists `README.md` and
  `LICENSE`, the release note says to keep them mirrored with the siblings and
  to bump the release number the README states, and the reference-index rule
  now counts three places to update — the *Quick start* list and *Reference
  directory* table in `SKILL.md` plus the *What's inside* table in `README.md`.

## [0.1.0] - 2026-09-01

Initial release of the booking-kiosk skill, extracted from a production
venue-booking management system — Next.js 16 App Router, React 19, Firestore,
Stripe — where the kiosk ran as one of five surfaces sharing a booking engine
at a multi-location venue. It teaches an agent to build a self-service
touchscreen booking terminal in someone else's codebase: a seven-step walk-up
flow with an on-screen keyboard, inactivity auto-reset, pay-at-counter or
pay-by-QR, live availability refresh, and a find-my-booking edit flow. The
templates are the hardened version of the source, not a copy of it.

### Added
- `SKILL.md` entry point: the frontmatter trigger, when to use and when not to,
  the architecture diagram, six critical facts, five hard rules, a five-step
  quick start, the Adaptation Contract table that bounds what a host must
  supply, and the reference directory table.
- `references/state-machine.md` — the canonical vocabulary and rename table,
  state shape, actions, the reducer and provider, the rules an orchestrator
  must follow, and an inline suite of 12 reducer tests. Carries the module's
  signature distinction: `SET_SLOT` navigates, `REFRESH_SLOT` never does.
- `references/screens.md` — the seven-step flow plus the two side-flow steps,
  the per-screen contract, payment-method resolution, the on-screen keyboard,
  the inactivity timer, touch hardening, modal semantics, and the canonical
  dictionary key tree in which every key is required.
- `references/api-contract.md` — the four kiosk routes, device auth, the single
  error envelope, create / lookup / add-items / edit with their guard tables,
  and the one-fetch-wrapper client rule.
- `references/booking-backend.md` — the `KioskBackend` interface as the only
  data seam, the capacity transaction, the two stock models, payments and
  failure modes, a Firestore reference implementation as audited in the source,
  and a relational sketch as a port target.
- `references/realtime-offline.md` — the `lastBookingChangeAt` freshness
  signal, availability fetching and caching, offline failover against a LAN
  server, and what deliberately stays out.
- `references/operations.md` — launching the kiosk, access gating, the operator
  surface, the canonical environment variable list, and a post-deploy smoke
  test.
- `references/provenance.md` — the audit ledger: 12 numbered source defects
  fixed in the templates, 8 choices kept deliberately with the reason each is
  safe, and 7 additions designed here but never run in production.
- `CLAUDE.md` with the repository's editing rules and content invariants, and
  `.gitignore`.

### Fixed
Twelve defect classes from the source module, each documented in
`references/provenance.md`. The five that became hard rules:
- All four kiosk routes rejected only a *wrong* device key — omitting the
  header passed, and the client never sent one, so unauthenticated internet
  callers could create and mutate bookings and consume stock. Now a configured
  key means a missing header is a 401.
- Slot and consumable prices arrived in the request body and flowed into the
  total and the payment amount, so any caller could book for zero. Prices are
  off the wire inbound; the server resolves them from its catalog and the
  client total is a preview.
- The silent availability refresh re-dispatched the navigating `SET_SLOT`, so a
  booking made by anyone else yanked a kiosk user off the summary screen and
  discarded their progress; selected stations never reconciled against
  refreshed availability. Now a data-only `REFRESH_SLOT` that drops taken
  stations, keeps participant names index-aligned, and never navigates.
- A capacity conflict returned 409 with the just-taken stock locks still ACTIVE
  for the full 15-minute TTL, the edit route released old locks before
  acquiring new ones, and the payment webhook released locks for failed
  add-item payments but not for failed bookings. Locks are now released on
  every failure path after acquisition.
- Every kiosk request goes through one fetch wrapper; the source's bare
  `fetch` calls broke silently under offline failover.

Also fixed: the client `sessionId` is now a server-checked idempotency key
(the source generated one nobody read and minted lock keys from `Date.now()`,
which collides across terminals); an invalid promo is a 400 before any write
instead of a silent full-price charge, and post-commit usage recording no
longer 500s on a booking that already exists; the unauthenticated lookup is
rate-limited with masked contact fields instead of returning full PII for any
guessable 8-character code; edit re-checks capacity and keeps the promo, and
add-items applies deltas inside a transaction with event-id dedupe; the
capacity re-check reads a date-overlap window instead of every PENDING and
CONFIRMED booking for the location; online payment is a real QR rather than a
truncated URL rendered as text, and every text input opens the on-screen
keyboard rather than suppressing the native one with nothing in its place.
Plus a drawer of smaller correctness fixes and the operator and accessibility
gaps — heartbeat, dialog semantics, focus management, touch targets, contrast,
and dead dictionary keys — listed as entries 11 and 12 in the ledger.
