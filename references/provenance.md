# Provenance

Written by the engineers who have shipped this kiosk module. The earlier
implementation it was audited against ran on Next.js App Router, React,
Firestore and Stripe, as one of several surfaces sharing a booking engine at a
multi-location venue with a LAN fallback server. The state machine, screen
flow, timers, stock model, capacity transaction and offline failover are
production-proven there. **The templates are the hardened version**: a
three-pass audit (correctness, code quality, operator usability) found the
defects below, and the skill ships the fixes rather than the bugs.

The earlier implementation's domain vocabulary is replaced by the canonical
`service / station / equipment / consumable / compatibilityKey / participant`
throughout the skill; see the vocabulary table in
[state-machine.md](state-machine.md).

## Fixed in the templates

### 1. Device auth that a missing header bypassed
All four kiosk routes checked `if (sent && configured && sent !== configured)`
— only a *wrong* key was rejected; omitting the header passed, and the kiosk
client never sent one. Unauthenticated internet callers could create and
mutate bookings and consume stock.
**Shipped:** configured key ⇒ required header ([api-contract.md](api-contract.md)).

### 2. Client-priced Stripe charges
Slot and consumable prices arrived in the request body and flowed into the
total and the Stripe amount. Any caller could book for zero.
**Shipped:** prices removed from the wire; server resolves from its catalog
([api-contract.md](api-contract.md), [booking-backend.md](booking-backend.md)).

### 3. Online payment was a dead end
The confirmation screen rendered `QR: <first 50 chars of URL>...` as text
(a placeholder comment said "use a QR library in production"); the main-flow
customer had no way to pay online. Separately, three of the six text inputs
set `inputMode="none"` (suppressing the native keyboard) without opening the
on-screen keyboard — email/phone/promo entry and the whole booking-lookup
screen were unusable on touch-only hardware.
**Shipped:** QR requirement + keyboard-on-every-field rule ([screens.md](screens.md)).

### 4. Background refresh teleported the user
The silent availability refresh re-dispatched `SET_SLOT`, whose reducer arm
also navigates — a booking made by anyone else yanked a kiosk user from the
summary back to station selection, discarding progress. Selected stations
also never reconciled against refreshed availability, so a stolen station
could still be submitted.
**Shipped:** separate `REFRESH_SLOT` data-only action that drops taken
stations and keeps names index-aligned, with tests
([state-machine.md](state-machine.md)).

### 5. Stock reserved and never released on failure paths
A capacity conflict returned 409 with the just-taken stock locks still
ACTIVE for the full 15-min TTL; the edit route released old locks *before*
acquiring new ones (a failure stranded the booking with no reservation); the
payment webhook released locks for failed add-item payments but not for
failed bookings.
**Shipped:** release-on-failure in the route skeleton; webhook failure path
releases booking locks ([api-contract.md](api-contract.md),
[booking-backend.md](booking-backend.md)).

### 6. No idempotency
The client generated a `sessionId` nobody ever read; the server minted lock
session keys from `Date.now()` (collides across terminals); the only
duplicate-submit guard was a client boolean. Retries and double-taps made
double bookings.
**Shipped:** `sessionId` as a server-checked idempotency key
([api-contract.md](api-contract.md)).

### 7. Promo failures were silent, then fatal
An invalid/expired/wrong-location code was ignored — the customer silently
paid full price. And usage-limit increments ran *after* booking commit
without a catch, so hitting the limit returned a 500 for a booking that
existed.
**Shipped:** invalid promo is a 400 before any write; post-commit usage
recording is caught ([api-contract.md](api-contract.md)).

### 8. PII on an unauthenticated, unthrottled lookup
`GET …/lookup?shortId=` returned full name, email, phone and participants
for any guessable 8-char code (31-char alphabet), min query length 3, with
no rate limit — an enumeration target.
**Shipped:** rate limit + masked contact fields ([api-contract.md](api-contract.md)).

### 9. Edit/add-items trusted and raced
The edit route re-wrote slots with **no capacity re-check** (could
double-book a station), recalculated pricing **without the promo** (silently
un-discounting on every edit), and skipped the module-block gate its
siblings had. Add-items had no cancelled-booking guard, mutated pricing in a
read-modify-write outside a transaction (concurrent adds lost increments),
appended lines via a set-union that deduplicated identical purchases while
still charging for both, and its webhook wrote back absolute totals computed
minutes earlier — with no event-id dedupe on redelivery.
**Shipped:** the guard table and delta-in-transaction rules
([api-contract.md](api-contract.md), [booking-backend.md](booking-backend.md)).

### 10. Unbounded transactional read
The capacity re-check read *every* PENDING/CONFIRMED booking for the
location — no date filter, no limit — inside every create transaction. Grows
forever; eventually every sale times out.
**Shipped:** date-overlap filter as part of the transaction spec
([booking-backend.md](booking-backend.md)).

### 11. A drawer of smaller correctness fixes
Back button inert on the side-flow steps (they were outside `STEP_ORDER`);
participant seeding clobbering a hand-edited contact name; cart surviving a
location switch with the old location's item ids; the calendar's "today"
memoized once (wrong after midnight); a 60 s clock under seconds-granularity
UI and slot cutoffs; the summary's payment method captured once and stale
after connectivity changes; a retry timer firing after unmount; no request
aborts anywhere; `catch(() => {})` on the pricing/equipment/locations
fetches (symptom: submit disabled with no reason shown); raw `Error.message`
in 500 bodies; two error-envelope shapes (client showed blank errors for
one); localhost as the default payment-redirect base URL; inactivity ref
written during render; silent per-lock confirm failures; equipment lines
silently skipped when unresolvable. Each fix lives in the reference that owns
the topic.

### 12. Operator and accessibility gaps
No remote heartbeat (a dead kiosk was found by walking past), no
stock-confirm-failure surfacing, modal without dialog semantics or focus
management, globally stripped focus outlines, icon-only buttons without
labels, 32 px touch targets on the busiest screen, failing contrast on hint
text, hardcoded source-language strings in shared hooks and persisted values,
dead dictionary keys, and an edit endpoint whose UI trigger was wired but
never called. **Shipped:** rules in [screens.md](screens.md) and the operator
table in [operations.md](operations.md); heartbeat is documented as an
extension, not shipped code.

## Kept deliberately

- **In-memory session state, no persistence** — an abandoned session must
  not show one customer's data to the next. Do not add localStorage.
- **Inline errors, never toasts** — arm's-length single-focus UI.
- **Fire-and-forget confirmation email** (now with `.catch`) — the customer
  is standing there; mail latency must not block the confirmation screen.
- **Counter bookings confirm immediately** (`PENDING_COUNTER_PAYMENT`) —
  the no-payment-integration path is the kiosk's resilience story.
- **Per-item stock-lock transactions** rather than one giant transaction —
  locks are independent; the booking create is the single serialization
  point. Proven under real concurrency.
- **A shared device key, not per-device tokens** — right-sized for one
  venue's LAN; the upgrade path is documented in
  [api-contract.md](api-contract.md).
- **Prop-drilled dictionary slices, no client i18n runtime** — one page,
  server-resolved locale; a runtime would be dead weight.
- **Hidden scrollbars, dark high-contrast chrome, hand-rolled modal** —
  kiosk-appropriate choices, kept as intent with accessibility requirements
  attached.

## Added (designed in the skill, never run in the earlier implementation)

- `REFRESH_SLOT` reconciliation semantics and the reducer test suite.
- Idempotency-key round trip.
- Keyboard `mode` variants (email/code) and the dictionary-driven accent row
  (the earlier implementation had a single QWERTY layout plus one hardcoded
  accent row).
- Lookup PII masking and rate limiting.
- Location-selector empty state.
- The relational backend sketch in [booking-backend.md](booking-backend.md).
- Stale-PENDING cleanup tightened from the earlier implementation's 25 h to
  1-2 h, a deliberate disagreement; a kiosk QR no-show holding stations all day is
  worse than an occasional slow payer.
