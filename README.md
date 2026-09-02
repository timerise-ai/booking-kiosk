# booking-kiosk

An [Agent Skill](https://agentskills.io) that teaches an agent to build a self-service touchscreen booking
kiosk in a **Next.js App Router** app: a seven-step walk-up flow from service, date and time slot through
stations and participant names, equipment and consumables, to summary and confirmation, with an on-screen
keyboard, inactivity auto-reset, pay-at-counter or pay-by-QR, live availability refresh, and a find-my-booking
edit flow. Backend- and payment-provider-agnostic.

A kiosk is a booking website with the trust model inverted. The client is unauthenticated by design and
physically shared by strangers, so **all trust lives on the server**, meaning prices, capacity, promo validity
and idempotency, and **all state is disposable**: in memory, auto-reset, never persisted, because the next
customer must not see the previous one's name. Everything in this skill follows from those two facts.

It was written by the engineers who have shipped this kiosk module. The earlier implementation it was audited
against ran as one of several surfaces sharing a booking engine at a multi-location venue with a LAN fallback
server. The templates carry its state machine, screen flow, timers, stock model, capacity transaction and
offline failover, and hold the properties a kiosk has to hold: every price, capacity check and promo decision
is computed on the server; every submit is idempotent; a configured device key makes a missing header a 401;
a stock lock is released on every failure path; a session lives in memory and resets on inactivity. Each one
is stated as what the reducer suite and the route guard tables verify.
[`references/provenance.md`](references/provenance.md) is the record of what the audit changed, what was kept
deliberately, and what is new in the skill.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/booking-kiosk
```

Name the agents instead with `-a`, for example `npx skills add timerise-ai/booking-kiosk -a claude-code -a codex`.

Or clone it yourself. Nothing here is Claude-specific: the skill is a plain [Agent
Skills](https://agentskills.io) folder, `SKILL.md` plus markdown references with no file that calls a model,
so cloning it into an agent's skills directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/booking-kiosk.git ~/.claude/skills/booking-kiosk
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/booking-kiosk ~/.agents/skills/booking-kiosk
```

Update the skill with `git pull` in its directory. The current release is **0.1.4**. See
[`CHANGELOG.md`](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: a venue (gym, karting, bowling,
climbing, escape room, clinic) wanting an unattended booking terminal, adding kiosk mode to an app that
already books online, or auditing an existing kiosk; also on phrases like "kiosk mode", "self-service kiosk",
"touch screen booking", "walk-up terminal", "on-screen keyboard", "counter payment",
`PENDING_COUNTER_PAYMENT`. Invoke it explicitly with `/booking-kiosk` in Claude Code, `$booking-kiosk` in
Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: when to use and when not to, the architecture diagram, six critical facts, five hard rules, the quick start, the Adaptation Contract table, and the reference directory |
| `references/state-machine.md` | Canonical vocabulary and rename table, state shape, actions, the reducer and provider, the rules an orchestrator must follow, and 12 inline reducer tests |
| `references/screens.md` | Screen flow, the per-screen contract, payment-method resolution, the on-screen keyboard, the inactivity timer, touch hardening, modals, and the dictionary key tree |
| `references/api-contract.md` | Route surface, device auth, the single error envelope, create / lookup / add-items / edit with their guard tables, and the one-fetch-wrapper client rule |
| `references/booking-backend.md` | The `KioskBackend` interface, the capacity transaction, the two stock models, payments, failure modes, a Firestore reference implementation, and a relational sketch |
| `references/realtime-offline.md` | The freshness signal, availability fetching and caching, offline failover against a LAN server, and what stays out |
| `references/operations.md` | Launching the kiosk, access gating, the operator surface, environment variables, and the post-deploy smoke test |
| `references/provenance.md` | The engineering ledger: what the audit of the earlier implementation changed and how the templates verify it, what was kept deliberately, and what is new in the skill |

The seam contract lives in the **Adaptation Contract** table in `SKILL.md`: it bounds what the host app must
supply: its domain vocabulary, location model, device-key management, ORM or SDK behind `KioskBackend`,
payment provider, realtime channel, LAN fallback, UI primitives, design system, i18n files and validation
library. If a template starts requiring something new from the host, it belongs in that table or it does not
belong at all. The offline half of the story is deliberately out of scope here: the on-prem fallback server is
the sibling [`island-mode-server`](https://github.com/timerise-ai/island-mode-server) skill, and this skill
defines only the kiosk's client-side failover contract against it.

## The five non-negotiables

These travel with the module and are never optional. They are the hard rules in `SKILL.md`: each one is the
rule, the reason it holds, and what verifies it (`references/provenance.md` has the record behind each):

1. **Never persist kiosk session state.** In memory, 120 s inactivity reset, 30 s confirmation reset, so an
   abandoned session never shows one customer's name to the next. The reducer suite covers the reset paths.
2. **Never trust the client for prices, capacity, or promo validity.** The server computes every amount and
   answers an invalid promo with `PROMO_INVALID` instead of a full-price fallback, because a customer at a
   kiosk has no way to check the amount. The guard tables in `references/api-contract.md` state each check.
3. **One fetch wrapper for every kiosk request.** The wrapper carries the device header, the error envelope
   and the failover base URL, so auth, errors and offline failover apply to every request at once.
4. **If the device key is configured, a missing header is a 401.** Rejecting only a wrong key is the same as
   no auth. The device-auth guard in `references/api-contract.md` checks presence before value.
5. **Release stock locks on every failure path after acquiring them.** A 409 that keeps the reservation
   freezes stock for the full 15-minute TTL. The failure-mode table in `references/booking-backend.md` lists
   every path.

One more invariant is the module's signature: **`SET_SLOT` navigates, `REFRESH_SLOT` never does.** User taps
move the customer; background availability refreshes must not teleport them off the summary screen
mid-purchase.

Everything else is the host app's: vocabulary, backend, payment provider, UI primitives, i18n.

## Not this

| Not this | Use instead |
|---|---|
| The on-prem offline fallback server | The sibling [`island-mode-server`](https://github.com/timerise-ai/island-mode-server) skill; this one defines only the kiosk's client-side failover contract |
| Digital signage or display screens | The sibling [`digital-signage`](https://github.com/timerise-ai/digital-signage) skill: playback, no input |
| Staff-facing POS or admin booking tools | The host's authenticated admin; staff are trusted, and this skill's trust model is wrong for them |
| A plain booking website | The backend seam if useful; the kiosk machinery (keyboard, timers, touch hardening) is dead weight there |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build, lint or test step, and nothing here
runs locally: the reducer suite in `references/state-machine.md` imports `./kiosk-context`, which exists only
once the templates are copied into a host project. Claims in this skill are meant to be verifiable: if you
change a factual claim, say how you verified it, whether against the library, the docs, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason `references/provenance.md` records, and that ledger must stay truthful:
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