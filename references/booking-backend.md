# The booking backend seam

The kiosk does not own bookings — it is one sales channel into a booking
system shared with a website and staff walk-ins. This file defines the
contract the kiosk routes need from that system, the concurrency patterns
that make it safe, and the failure modes proven in production. The Firestore
notes reflect the audited earlier implementation; the relational notes are a port sketch,
**designed, not production-proven** — treat accordingly.

## The interface

```ts
export interface ExistingBooking { id: string; shortId: string; checkoutUrl?: string }

export interface PricedLine {
  itemId: string; name: string; unitPrice: number; quantity: number; lineTotal: number;
}

export interface NewBooking {
  idempotencyKey: string;
  locationId: string;
  serviceType: string;
  windows: Array<{ slotId: string; station: string; dateTimeFrom: string; dateTimeTo: string }>;
  pricing: { slotsTotal: number; consumablesTotal: number; subtotal: number;
             grandTotal: number; lines: PricedLine[]; discount: number; promoId?: string };
  contact: { fullName: string; email: string; phone: string };
  participants?: string[];
  parentBookingId?: string;
  paymentMethod: 'online' | 'counter';
  stockLockIds: string[];
  source: 'kiosk';
}

/** Host-shaped stored booking; the lookup route maps it to the client's
 *  LookupBooking with masked contact fields. */
export interface StoredBooking {
  id: string; shortId: string; locationId: string; status: string;
  contact: { fullName: string; email: string; phone: string };
  [key: string]: unknown;
}

export class CapacityError extends Error {
  constructor(
    public code: 'SLOT_UNAVAILABLE' | 'STATION_TAKEN' | 'NO_STATIONS',
    public reason: string,
  ) { super(reason); }
}
export const isCapacityError = (e: unknown): e is CapacityError => e instanceof CapacityError;

export interface KioskBackend {
  findByIdempotencyKey(key: string): Promise<ExistingBooking | null>;

  /** Resolve every price server-side from the catalog. Client prices do not exist. */
  resolvePricing(input: {
    locationId: string; serviceType: string; currency: string;
    slots: Array<{ slotId: string; station: string; dateTimeFrom: string; dateTimeTo: string }>;
    consumables: Array<{ itemId: string; quantity: number }>;
  }): Promise<
    | { ok: true; value: { slotsTotal: number; consumablesTotal: number;
                           subtotal: number; grandTotal: number;
                           lines: PricedLine[] } }
    | { ok: false; error: string }        // unknown itemId, currency not priced…
  >;

  validatePromo(code: string, ctx: { currency: string; locationId: string; subtotal: number }):
    Promise<{ valid: true; id: string; discountAmount: number } | { valid: false; reason: string }>;
  recordPromoUse(id: string): Promise<void>;

  lockStock(input: {
    locationId: string; sessionId: string; source: 'kiosk';
    slots: Array<{ dateTimeFrom: string; dateTimeTo: string }>;
    equipment?: Array<{ slug: string; inventoryItemId?: string; quantity?: number }>;
    consumables: Array<{ itemId: string; quantity: number }>;
  }): Promise<{ ok: true; ids: string[] }
            | { ok: false; detail: { item: string; requested: number; available: number } }>;
  confirmStockLocks(ids: string[], bookingId: string): Promise<void>;
  releaseStockLocks(ids: string[]): Promise<void>;

  createBookingTransactionally(input: NewBooking): Promise<{ bookingId: string; shortId: string }>;

  createCheckoutSession(input: {
    amount: number; currency: string; bookingId: string; shortId: string; locale: string;
  }): Promise<{ url: string }>;

  sendConfirmation(b: { bookingId: string; shortId: string }, locale: string): Promise<void>;

  findBookingByShortId(shortId: string): Promise<StoredBooking | null>;
}
```

Money is **integer minor units** end to end (grosze, cents). Convert to the
payment provider's expectations only at that boundary — zero-decimal
currencies (ISK, JPY) divide by 100 exactly there and nowhere else.

## Capacity: the one transaction that matters

Availability is computed, never stored: capacity per slot = sum of active
station resources, minus stations held by `PENDING`/`CONFIRMED` bookings in
the same window. The create path **re-checks inside the write transaction**:

```
transaction {
  existing = bookings WHERE locationId AND serviceType
                       AND status IN (PENDING, CONFIRMED)
                       AND window overlaps requested window        ← filter, see below
  for each requested window:
    reject if a requested station is already taken        → STATION_TAKEN
    reject duplicate stations within the request itself   → SLOT_UNAVAILABLE
    reject if free-station count < requested count        → NO_STATIONS
  insert booking
  bump location.lastBookingChangeAt                       ← the realtime signal
}
```

Two hard-won details:

- **Normalize station identity before comparing.** Different channels label
  the same physical station differently ("Court 1" vs "C1"). The earlier
  implementation normalized by trailing digits so cross-channel collisions were
  caught. Whatever your labels, define one canonical station key and compare
  on that.
- **Filter the transactional read by date.** The earlier implementation read *every*
  PENDING/CONFIRMED booking for the location — no date filter, no limit —
  inside every create transaction. That is unbounded growth: a successful
  venue eventually times out every sale. Query only bookings whose window can
  overlap the requested day.

On success, bump a `lastBookingChangeAt` timestamp on the location — this
single field drives every terminal's silent availability refresh
([realtime-offline.md](realtime-offline.md)).

`shortId` — the human-facing code on the confirmation screen — is 8 chars
from an unambiguous alphabet (no 0/O, 1/I). Allocate it inside the create
transaction or with a unique constraint; the earlier implementation checked uniqueness with a
separate pre-read, leaving a collision window.

## Stock: two models, do not mix them

| Model | For | Reserve | Confirm | Availability |
|---|---|---|---|---|
| **Consumable** | balls, tokens, credits | increment `reservedStock` | decrement `stockLevel`, write an audit log entry | `stockLevel - reservedStock` |
| **Time-bound** | equipment: rentable items that come back | create a lock with the slot window | mark lock CONFIRMED; `stockLevel` untouched (it is fleet size) | fleet size − locks overlapping the window |

Lifecycle: `lockStock` at submit (status ACTIVE, **TTL ~15 min**) →
`confirmStockLocks` on counter-confirm or payment webhook → a sweep job
releases expired ACTIVE locks (every 5 min in the earlier implementation). The TTL is what makes
abandoned kiosk sessions self-heal; the release-on-failure in the create
route ([api-contract.md](api-contract.md)) is what stops a capacity conflict
from freezing stock for the whole TTL anyway.

Confirm failures must not be silent. The earlier implementation wrapped each lock confirm in
a per-lock catch that only logged — a failed confirm meant stock never
decremented while the booking said CONFIRMED, and nobody was told. Fail loud:
retry once, then flag the booking for operator attention
([operations.md](operations.md)).

Equipment lines without a resolvable inventory id were *skipped with a
console.warn* in the earlier implementation, so the booking confirmed with
unreserved equipment. Decide explicitly: either reject the line (strict) or record it as
unreserved on the booking so staff can see it. Silent is the only wrong
answer.

## Payments

### Counter (`PENDING_COUNTER_PAYMENT`)

The kiosk's superpower: no payment integration in the critical path. Booking
is created `CONFIRMED` with `payment.status = PENDING_COUNTER_PAYMENT`,
stock confirms immediately, the customer shows the shortId at the counter and
staff mark it paid in their panel. This path must keep working when the
internet is down ([realtime-offline.md](realtime-offline.md)).

### Online (Stripe Checkout, via QR)

Booking is created `PENDING`; stock locks stay ACTIVE; a Checkout session is
created with metadata `{ type: 'BOOKING', bookingDocId, shortId, source: 'kiosk' }`;
the kiosk shows the URL as a QR.

The **webhook** completes the sale: on `checkout.session.completed` →
confirm booking + confirm stock locks; on expiry/failure → fail the booking
**and release its stock locks** (the earlier implementation released locks on failure for
add-item payments but not for bookings — failed bookings held stock until the
TTL sweep).

Webhook rules proven necessary:

- **Idempotent by event id.** Stripe redelivers. The earlier implementation re-ran stock
  confirms and re-wrote totals on redelivery. Store processed event ids.
- **Deltas, not absolutes.** Apply changes by re-reading the booking in a
  transaction. Never stash computed totals in session metadata and write
  them back later — minutes may have passed.
- **Payment succeeded but capacity is gone** (someone else confirmed in the
  gap): do not un-charge silently. Flag the booking
  (`payment.requiresRefund = true`) and surface it to operators; refunds are
  a human decision.

### Cleanup jobs

| Job | Cadence | Action |
|---|---|---|
| Expired stock-lock sweep | 5 min | release ACTIVE locks past TTL |
| Stale PENDING bookings | hourly | fail bookings whose payment never completed. Source used 25 h — far too long for a kiosk: a no-show QR holds stations all day. **1–2 h is right** (this tightening is a deviation from the earlier implementation; record disagreement if the venue wants otherwise). |

## Failure modes

| Event | Server behavior | Kiosk behavior |
|---|---|---|
| Station taken between tap and submit | 409 `STATION_TAKEN` from the create tx; locks released | `REFRESH_SLOT` usually catches it first and drops the station; else show the mapped error inline, stay on summary |
| Not enough stock | 409 `INSUFFICIENT_STOCK` + item/requested/available | interpolated inline message |
| Slot started while user dawdled | 400 `BOOKING_TIME_IN_PAST` | 1 s now-tick disables the slot first; error inline as backstop |
| Promo invalid/expired/exhausted | 400 `PROMO_INVALID` at validate **and** at create | inline error at the promo field |
| Payment webhook never arrives | stale-PENDING job fails the booking, releases locks | customer at the counter: staff look up the shortId and take payment manually |
| Paid but capacity lost | booking flagged for refund, operator alerted | — |
| Duplicate submit / lost response | idempotency key returns the original booking | at most one booking exists |
| Catalog fetch fails on kiosk | — | the submit button must say *why* it is disabled; the earlier implementation greyed it silently when pricing failed to load |

## Firestore reference (as audited in the earlier implementation)

Collections: `bookings` (embedded cart, pricing, contact, `stockLockIds`,
`bookingShortId`, `source: 'kiosk'`), `inventory` (slots/equipment/
consumables with `stockLevel`, `reservedStock`), `stockLocks`, `locations`
(with `lastBookingChangeAt`), `promoCodes`, `inventoryLogs` (audit trail).
The capacity re-check runs in `runTransaction`; each stock lock and confirm
is its own transaction (acceptable: locks are per-item independent; the
booking tx is the serialization point). Add a composite index for the
date-filtered capacity read and an index on the idempotency-key field.

## Relational sketch (port target — not production-proven)

`bookings`, `booking_slots` (one row per station×window with a unique
constraint on `(location_id, station_key, starts_at)` — the constraint *is*
the capacity check for station collisions; count-based capacity still needs
`SELECT … FOR UPDATE` on the window), `stock_locks` with `expires_at`,
`idempotency_key UNIQUE` on bookings. Everything in one transaction: lock
rows → insert → notify (`NOTIFY`/`LISTEN` or the host's realtime layer
replaces `lastBookingChangeAt`).
