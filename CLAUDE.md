# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **Claude Skill package**, not an application. It ships prose + TypeScript
templates that teach another agent how to build a self-service booking kiosk in
*someone else's* Next.js codebase. There is nothing here to run — no
`package.json`, no build, no lint, no dev server. The version of record is the
git tag; there is no version field anywhere in the tree.

```
SKILL.md                  entry point: frontmatter trigger + critical facts + hard rules + routing table
references/*.md           6 topic files + provenance.md, loaded on demand by the routing table
CHANGELOG.md              Keep a Changelog; one section per git tag
README.md                 human-facing: install, contents table, the five non-negotiables
LICENSE                   MIT, identical to the siblings'
```

Sibling directories under `../` (`island-mode-server`, `digital-signage`,
`help-center-markdown`, …) are other skills, not dependencies.
`island-mode-server` is referenced by name from `SKILL.md` and
`references/realtime-offline.md`.

## Commands

None. The reducer test suite lives inline in
`references/state-machine.md` and **cannot run here** — it imports
`./kiosk-context`, which exists only after the templates are copied into a host
project. Do not add a `package.json` or a test config to make it runnable
locally; verify it by reading it against the reducer template in the same file,
or by running it in a target project after adaptation.

Releases are cut with `/bumpv`: it writes the `CHANGELOG.md` section and tags
`vX.Y.Z`, matching the siblings' convention — Keep a Changelog + SemVer, git
tags as the version of record, no version field in the SKILL.md frontmatter,
and release commits carrying no AI attribution. `README.md` and `LICENSE`
mirror the siblings; keep them that way rather than inventing new conventions.
The README names the current release number — bump it with the tag.

## Editing rules specific to this repo

**SKILL.md is a router, not a tutorial.** It carries only what an agent must
know before choosing a reference: the frontmatter trigger, the when/when-not
sections, the architecture diagram, the 6 critical facts, the 5 hard rules, the
Adaptation Contract table, and the routing table. Deep material belongs in
`references/`. Keep it short — it is loaded on every trigger; references are
not.

**Three places index the references.** Adding, renaming, or splitting a file in
`references/` means updating the *Quick start* numbered list and the *Reference
directory* table (including its trigger keywords) in SKILL.md, the *What's
inside* table in README.md, plus any cross-links in sibling references.
Cross-links between references are relative and sibling-style
(`[screens.md](screens.md)`); links from SKILL.md are `references/`-prefixed.

**`references/provenance.md` is the audit ledger and must stay truthful.** It
is the only place that distinguishes what ran in the earlier implementation
from what was designed here. Keep it free of the earlier implementation's
own paths, component names and domain terms. Any change to a template updates it:
- fixing a newly found defect: a numbered entry under *Fixed in the templates*
- keeping a questionable earlier choice: an entry under *Kept deliberately*
  with the reason it is safe
- inventing something the earlier implementation never ran: an entry under
  *Added*

Its numbered ledger is referenced by count elsewhere; the 12-entry list and the
inline suite of **12 `it()` tests in one `describe('kioskReducer')` block** in
`references/state-machine.md` are concrete claims — changing the suite means
changing any statement of its size.

**Do not weaken the five hard rules** in SKILL.md (no session persistence, no
client-trusted prices/capacity/promo, one fetch wrapper, configured key ⇒
required header, release stock locks on every failure path). They correspond to
entries 1, 2, 5 and 6 in the provenance ledger and to the *Kept deliberately*
entries: each holds for a reason the ledger records, not as a stylistic
preference.

## Content invariants the templates depend on

These recur across references; changing one requires sweeping all of them.

- **Fixed domain vocabulary.** Templates always say
  `service / station / equipment / consumable / compatibilityKey / participant`
  and scope by `locationId`. The rename table lives in two places — the
  *Canonical vocabulary* table in `references/state-machine.md` and the
  vocabulary note in `references/provenance.md`. The host renames at adoption
  time — never rename inside the skill to match a particular product, and do
  not reintroduce the earlier implementation's own domain terms.
- **The Adaptation Contract table in SKILL.md is the full boundary** of what a
  host must supply. If a template starts requiring something new from the host,
  it goes in that table or it does not belong.
- **`SET_SLOT` navigates, `REFRESH_SLOT` never does.** Any new action added to
  `references/state-machine.md` must land on one side of that line explicitly;
  collapsing them back together is the module's signature bug.
- **The seven-step `STEP_ORDER`** (`service-select`, `date-select`,
  `time-select`, `participants`, `equipment`, `summary`, `confirmation`) plus
  the two side-flow steps (`booking-lookup`, `booking-edit`) must stay in sync
  across the reducer in `state-machine.md`, the flow diagram and screen
  contract table in `screens.md`, and the diagram in SKILL.md. `GO_BACK` must
  keep working on the side-flow steps, which are outside `STEP_ORDER`.
- **Every screen string is a required dictionary key.** The key tree in
  `references/screens.md` is canonical; a screen that reads a key not in the
  tree, or a tree key no screen reads, is an error.
- **The `KioskBackend` interface in `references/booking-backend.md` is the only
  data seam.** Routes in `api-contract.md` call nothing else; new backend
  capability means a new method on that interface.
- **Money is integer minor units end to end**, converted only at the payment
  provider boundary. No price ever crosses the wire inbound.
- **Timer values are fixed constants, verified on hardware**: 120 s
  inactivity, then the dim overlay and `RESET`; 30 s confirmation auto-reset;
  15 min stock-lock TTL; 5 s health poll with a 3 s abort and 3 consecutive
  failures before flipping offline; 1 to 2 h stale-PENDING cleanup. They
  appear in `state-machine.md`, `screens.md`, `realtime-offline.md` and
  `booking-backend.md`: change one, change all, and say so in `provenance.md`.
- **Env var names** in `references/operations.md` are the canonical list
  (`KIOSK_API_KEY`, public base URL, payment keys + webhook secret, LAN
  fallback URL). Reference them by the same names in `api-contract.md` and
  `realtime-offline.md`.
- **Framework claims.** Next.js App Router route handlers and React context are
  the shipped shape, but backend, payment provider, realtime channel and design
  system are all stated as substitutable; keep new templates'
  framework-specific surface thin enough that the claim holds.
