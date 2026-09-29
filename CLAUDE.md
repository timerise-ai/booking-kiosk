# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here, no build,
no lint, no dev server, and nothing in this repository executes. It teaches an agent how to build a
self-service touchscreen booking kiosk in *someone else's* **Next.js App Router** codebase: a stepped walk-up
flow, an on-screen keyboard, inactivity reset, pay-at-counter or pay-by-QR, live availability, and a
server-priced, idempotent booking API behind one backend seam.

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The reducer suite in `references/state-machine.md` imports `./kiosk-context`, which exists
only after the templates are copied into a host project, so it **cannot run here**. Do not add a
`package.json` or a test config to make it runnable locally; verify it by reading it against the reducer
template in the same file, or by running it in a target project after adaptation.

The skill was written by the engineer who has shipped this module; the earlier implementation it was audited
against was a venue kiosk, one of several surfaces sharing a booking engine, with a LAN fallback server.
`references/provenance.md` is the ledger of that audit: seventeen numbered entries on what changed and how the
templates verify it, what was kept deliberately, and what was designed here and has never run in production.
That file is the rationale layer: read it before "simplifying" anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. It is a router, not a tutorial: the frontmatter `description` is the trigger
  surface; the body carries when and when not to use, the architecture diagram, six **critical facts**, five
  **hard rules**, the quick start, the **Adaptation Contract** table (this skill's seam contract, kept here
  instead of a `references/adaptation.md`), the **reference directory** table mapping trigger keywords to
  files, and a closing line linking the skills index. Deep material belongs in `references/`.
- `README.md`: the human-facing front door, in the section order of the skill standard: install,
  activation, the file table, the five non-negotiables, the *Not this* table, contributing.
- `references/*.md`: one topic per file, loaded on demand. `state-machine.md` (vocabulary, reducer, the
  14-test suite), `screens.md` (flow, screen contract, keyboard, timers, dictionary key tree),
  `api-contract.md` (routes, device auth, guard tables, the fetch wrapper), `booking-backend.md` (the
  `KioskBackend` seam, capacity transaction, stock, payments), `realtime-offline.md` (freshness signal,
  offline failover), `operations.md` (launch, gating, env vars, smoke test), `provenance.md` (the audit).
- `CHANGELOG.md`: Keep a Changelog, one section per git tag. The git tag is the version of record; there is
  no version field in the SKILL.md frontmatter. `LICENSE` is MIT, identical to the siblings'.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt
  is the agent eval run before every release. Every other file there is one eval run: measured frontmatter
  that is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that
  has results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, run on every
  published release and on a maintainer's dispatch. It is copied verbatim from the standard and is the same
  in every skill; do not edit it, and never add a trigger on `push` or `pull_request`.

Sibling directories under `../` (`island-mode-server`, `digital-signage`, `help-center-markdown` and the
rest) are other skills, not dependencies. `island-mode-server` is referenced by name from `SKILL.md` and
`references/realtime-offline.md`.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment. A continuation block that extends
  a file already introduced omits it. Keep imports complete and types explicit.
- **Identifiers are shared across files.** `KioskBackend`, `KioskStep`, `STEP_ORDER`, `kioskReducer`,
  `SET_SLOT`, `REFRESH_SLOT`, `lastBookingChangeAt`, `shortId`, `sessionId`, `PROMO_INVALID`,
  `PENDING_COUNTER_PAYMENT` and the `x-kiosk-api-key` header appear in several references. Rename in all of
  them or none.
- **Keep the three tables in sync** with `references/`: the reference directory (with its trigger keywords)
  and the quick-start list in `SKILL.md`, and the *What's inside* table in `README.md`, plus any cross-links.
  Links are relative: `[x.md](references/x.md)` from `SKILL.md`, `[x.md](x.md)` between references.
- **`references/provenance.md` must stay truthful.** It is the only place that distinguishes what ran in the
  earlier implementation from what was designed here; keep it free of that implementation's own paths,
  component names and domain terms. Any change to a template updates it: a newly found defect gets a
  numbered entry under *Fixed in the templates*, a questionable earlier choice kept gets an entry under
  *Kept deliberately* with the reason it is safe, and anything the earlier implementation never ran goes
  under *Added*.
- **The odd-looking parts stay.** The data-only `REFRESH_SLOT`, the presence check before the key compare,
  the lock release on a 409, the same `notFound` for a missing and a foreign booking, the masked lookup
  fields: each is a ledger entry. Check `provenance.md` before simplifying one.
- **The numbers that remain are load-bearing.** Seventeen ledger entries; **14 `it()` tests in one
  `describe('kioskReducer')` block**; 120 s inactivity, then the dim overlay and `RESET`; 30 s confirmation
  auto-reset; 15 min stock-lock TTL; 5 s health poll with a 3 s abort and 3 consecutive failures before
  flipping offline; 1 to 2 h stale-PENDING cleanup. The timers are verified on hardware and appear in
  `state-machine.md`, `screens.md`, `realtime-offline.md` and `booking-backend.md`: change one, change all,
  and say so in `provenance.md`. Changing the suite means changing every statement of its size. Figures
  describing the earlier implementation's deployment do not appear anywhere.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier
  implementation is stated as such, in the *Added* section of `provenance.md` or marked *(extension)* where
  it appears.
- **Never present the non-negotiables as optional.** The five hard rules in `SKILL.md` (no session
  persistence, no client-trusted prices, capacity or promo, one fetch wrapper, a configured key means a
  required header, stock locks released on every failure path) are the README's five non-negotiables, in
  the same order. They correspond to entries 1, 2, 5 and 6 in the ledger and to its *Kept deliberately*
  entries.
- **`SET_SLOT` navigates, `REFRESH_SLOT` never does.** Any new action in `state-machine.md` lands on one side
  of that line explicitly; collapsing them back together is the module's signature bug.
- **The seven-step `STEP_ORDER`** (`service-select`, `date-select`, `time-select`, `participants`,
  `equipment`, `summary`, `confirmation`) plus the two side-flow steps (`booking-lookup`, `booking-edit`)
  stay in sync across the reducer in `state-machine.md`, the flow diagram and screen contract table in
  `screens.md`, and the diagram in `SKILL.md`. `GO_BACK` keeps working on the side-flow steps, which are
  outside `STEP_ORDER`.
- **Every screen string is a required dictionary key.** The key tree in `screens.md` is canonical; a screen
  that reads a key not in the tree, or a tree key no screen reads, is an error.
- **`KioskBackend` in `booking-backend.md` is the only data seam.** Routes in `api-contract.md` call nothing
  else; new backend capability means a new method on that interface. The **Adaptation Contract** table in
  `SKILL.md` is the full boundary of what a host supplies: a template that needs something new from the host
  adds a row there or does not belong.
- **Money is integer minor units end to end**, converted only at the payment provider boundary. No price
  ever crosses the wire inbound.
- **Env var names** in `operations.md` are the canonical list (`KIOSK_API_KEY`, `KIOSK_PUBLIC_BASE_URL`,
  `KIOSK_LAN_FALLBACK_URL`, and the payment provider's own pair, `STRIPE_SECRET_KEY` and
  `STRIPE_WEBHOOK_SECRET`); `api-contract.md`, `realtime-offline.md` and the quick start in `SKILL.md` use the
  same names, and the `.env.example` rule there lists the same three.
- **Framework claims.** Next.js App Router route handlers and React context are the shipped shape; backend,
  payment provider, realtime channel and design system are stated as substitutable, so keep a new template's
  framework-specific surface thin enough that the claim holds.
- **Which names the host renames.** The domain vocabulary
  (`service / station / equipment / consumable / compatibilityKey / participant`, scoped by `locationId`) is
  renamed by the host at adoption time through the *Canonical vocabulary* table in `state-machine.md`, with
  a matching note in `provenance.md`. Never rename it inside the skill to match a product. Route paths, the
  action names, the error codes and the env var names are the authoring contract and are not renamed.
- **Evals are not skill content.** A new prompt or an eval result is committed as `chore(evals): ...`,
  never causes a version bump and never rides in a release commit. A failing run stays committed.
- **Releases** follow section 9 of the index's STANDARD.md: `/bumpv` writes the `CHANGELOG.md` section, the
  README's current-release line moves with it, the release commit `chore(release): X.Y.Z` carries only
  those two, and the tag is `vX.Y.Z`. `README.md` and `LICENSE` mirror the siblings; keep them that way. No
  file and no commit carries AI or tool attribution.
- **Plain punctuation.** No em-dashes, en-dashes, arrows, middle dots or smart quotes in any markdown, code
  comments and diagrams included; prose wraps at 110 columns.
