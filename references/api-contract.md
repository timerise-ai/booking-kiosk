# Kiosk API contract

Four kiosk-owned endpoints plus the shared read endpoints the screens consume.
The kiosk client is unauthenticated by nature — anyone standing at the screen
is a legitimate user — so the server, not the client, is where every trust
decision lives.

## Route surface

| Route | Method | Purpose | Auth |
|---|---|---|---|
| `/api/kiosk/booking/create` | POST | New booking, counter or online payment | device key |
| `/api/kiosk/booking/lookup?shortId=` | GET | Find booking by short code (edit flow) | device key + rate limit |
| `/api/kiosk/booking/add-items` | POST | Append consumables to an existing booking | device key |
| `/api/kiosk/booking/edit` | POST | Update participants / contact on a booking | device key |
| `/api/pricing/get?locationId=` | GET | Price catalog (slots + consumables) | public read |
| `/api/inventory/equipment?locationId=&from=&to=` | GET | Equipment available in a window | public read |
| `/api/booking/slots?…` | GET | Month + day availability | public read, CDN-cached |

## Device auth

One shared header, `x-kiosk-api-key`, checked like this:

```ts
export function checkKioskKey(req: Request): boolean {
  const configured = process.env.KIOSK_API_KEY;
  if (!configured) return true;               // explicit opt-out: LAN-only / demo
  return req.headers.get('x-kiosk-api-key') === configured;
}
```

**When the key is configured, a missing header is a 401.** The earlier implementation checked
`if (sent && configured && sent !== configured)` — i.e. only a *wrong* key was
rejected and omitting the header bypassed auth entirely, on all four routes,
while the kiosk client never sent the header at all. Configure the key, send
it from a small `kioskFetch` wrapper (server-injected, not `NEXT_PUBLIC_`
where you can avoid it), and treat the no-key mode as a conscious deployment
decision, not a fallback.

A shared device key is still a shared secret: it identifies "a kiosk", not
*which* kiosk, and cannot be revoked per-device. Good enough for one venue's
LAN; if kiosks live on the public internet, issue per-device tokens
(documented extension, not shipped; the sibling display module did
per-device pairing tokens and it is the right upgrade path).

## Error envelope

Every response, success or failure, uses one shape. The earlier implementation
mixed two and the client's error branch showed blank messages for one of them:

```ts
export type ApiOk<T> = { ok: true; data: T };
export type ApiErr   = { ok: false; error: string; code?: string;
                         item?: string; requested?: number; available?: number };
```

```ts
export function ok<T>(data: T, status = 200): Response {
  return Response.json({ ok: true, data }, { status });
}
export function err(status: number, error: string,
             extra?: Partial<Omit<ApiErr, 'ok' | 'error'>>): Response {
  return Response.json({ ok: false, error, ...extra }, { status });
}
```

`error` is a stable machine-ish string; `code` is what the client maps to a
dictionary key for display. Never put raw `Error.message` from a caught
exception into `error` on a 500: the earlier implementation leaked internal
messages (collection names, stack fragments) to an unauthenticated endpoint.

Codes the client must handle: `SLOT_UNAVAILABLE`, `STATION_TAKEN`,
`NO_STATIONS`, `INSUFFICIENT_STOCK` (with `item`/`requested`/`available`
interpolated into the message), `BOOKING_TIME_IN_PAST`, `PROMO_INVALID`,
`STRIPE_DISABLED`, `PAYMENT_REQUIRED` (402, deployment/module blocked).

## Create

Request:

```ts
export interface KioskCreateRequest {
  idempotencyKey: string;            // the client sessionId — see below
  serviceType: string;
  locationId: string;
  locale: string;
  currency: string;
  fullName: string;
  email?: string;
  phone?: string;
  paymentMethod: 'online' | 'counter';
  promoCode?: string;
  participants?: string[];
  parentBookingId?: string;          // set when adding a slot to an existing booking
  cart: {
    slots: Array<{
      slotId: string;
      station: string;
      dateTimeFrom: string;          // ISO
      dateTimeTo: string;
    }>;
    equipment?: Array<{ slug: string; inventoryItemId?: string; quantity?: number }>;
    consumables: Array<{ itemId: string; quantity: number }>;
  };
}
```

Two deliberate absences versus the earlier implementation:

- **No prices anywhere in the request.** The earlier implementation sent `price` per slot and
  per consumable and the server fed them straight into the total — and into
  the Stripe charge amount. A tampered request could buy anything for zero.
  The server resolves every price from its own catalog by id
  ([booking-backend.md](booking-backend.md)); the client's displayed total is
  a preview, and a mismatch between preview and charge is a bug to surface,
  not reconcile silently.
- **No redundant `slotIds` array.** The earlier implementation sent N copies of the same
  slotId alongside the cart and the server never read them.

Response `201`: `{ ok: true, data: { bookingId, shortId, checkoutUrl? } }`.
`checkoutUrl` present iff `paymentMethod === 'online'` — render it as a QR
([screens.md](screens.md)).

### Validation

Dependency-free guards (swap for the host's schema library if it has one —
that is the validation seam). Never `as KioskCreateRequest` a parsed body:
the earlier implementation did, and a missing `cart` field became a 500 with a leaked stack
message.

```ts
const MAX_SLOTS = 20;
const MAX_LINES = 50;
const MAX_PARTICIPANTS = 40;
const NAME_MAX = 120;

function isNonEmptyString(v: unknown, max = 500): v is string {
  return typeof v === 'string' && v.trim().length > 0 && v.length <= max;
}
function isQuantity(v: unknown): v is number {
  return typeof v === 'number' && Number.isInteger(v) && v > 0 && v <= 1000;
}
function isIsoDate(v: unknown): v is string {
  return typeof v === 'string' && !Number.isNaN(Date.parse(v));
}

export function parseCreateRequest(body: unknown): KioskCreateRequest | { error: string } {
  if (typeof body !== 'object' || body === null) return { error: 'invalid_body' };
  const b = body as Record<string, unknown>;
  if (!isNonEmptyString(b.idempotencyKey, 100)) return { error: 'idempotencyKey_required' };
  if (!isNonEmptyString(b.serviceType, 50)) return { error: 'serviceType_required' };
  if (!isNonEmptyString(b.locationId, 100)) return { error: 'locationId_required' };
  if (!isNonEmptyString(b.fullName, NAME_MAX)) return { error: 'fullName_required' };
  if (b.paymentMethod !== 'online' && b.paymentMethod !== 'counter')
    return { error: 'paymentMethod_invalid' };
  if (!isNonEmptyString(b.currency, 3)) return { error: 'currency_invalid' };

  const cart = b.cart as Record<string, unknown> | undefined;
  const slots = Array.isArray(cart?.slots) ? cart.slots : null;
  if (!slots || slots.length === 0 || slots.length > MAX_SLOTS)
    return { error: 'slots_invalid' };
  for (const s of slots as Array<Record<string, unknown>>) {
    if (!isNonEmptyString(s.slotId, 100) || !isNonEmptyString(s.station, 100) ||
        !isIsoDate(s.dateTimeFrom) || !isIsoDate(s.dateTimeTo))
      return { error: 'slot_shape_invalid' };
  }
  const consumables = Array.isArray(cart?.consumables) ? cart.consumables : [];
  if (consumables.length > MAX_LINES) return { error: 'consumables_invalid' };
  for (const c of consumables as Array<Record<string, unknown>>) {
    if (!isNonEmptyString(c.itemId, 100) || !isQuantity(c.quantity))
      return { error: 'consumable_shape_invalid' };
  }
  const participants = Array.isArray(b.participants) ? b.participants : [];
  if (participants.length > MAX_PARTICIPANTS ||
      participants.some((p) => !isNonEmptyString(p, NAME_MAX)))
    return { error: 'participants_invalid' };

  return body as KioskCreateRequest;   // shape now proven field-by-field
}
```

Bounds matter as much as shapes: quantities are positive integers with a cap
(a negative quantity in the earlier implementation underflowed the total into a discount),
arrays are capped, and every string has a max length.

### Idempotency

The client sends its `sessionId` as `idempotencyKey`. Server-side:

```
1. Look up an existing booking where idempotencyKey == key (indexed field).
2. Found and < 10 min old → return it (200, same payload shape) — this is a
   retry of a request whose response was lost.
3. Not found → create, storing the key on the booking.
```

The earlier implementation's only duplicate protection was a client-side `submitting` boolean:
a tap racing a re-render, a network retry, or a flaky kiosk touch controller
produced double bookings. The server minted its own key from `Date.now()`,
which two terminals can share in the same millisecond.

### Route skeleton

The full order of operations, with backend calls behind the `KioskBackend`
seam ([booking-backend.md](booking-backend.md)):

```ts
export async function POST(request: Request) {
  if (!checkKioskKey(request)) return err(401, 'unauthorized');
  // Host seams: deployment payment-lock, module/feature gates go here.

  let raw: unknown;
  try { raw = await request.json(); } catch { return err(400, 'invalid_json'); }
  const parsed = parseCreateRequest(raw);
  if ('error' in parsed) return err(400, parsed.error);
  const body = parsed;

  const existing = await backend.findByIdempotencyKey(body.idempotencyKey);
  if (existing) return ok({ bookingId: existing.id, shortId: existing.shortId,
                            checkoutUrl: existing.checkoutUrl });

  // 1. Server-side pricing — the only prices that exist.
  const pricing = await backend.resolvePricing({
    locationId: body.locationId, serviceType: body.serviceType,
    slots: body.cart.slots, consumables: body.cart.consumables,
    currency: body.currency,
  });
  if (!pricing.ok) return err(400, pricing.error);

  // 2. Promo: an invalid code is an ERROR, not a silent no-op. The earlier
  //    implementation silently charged full price when a code was expired or
  //    wrong-location;
  //    at a kiosk the customer cannot see that happened until they pay.
  let discount = 0; let promoId: string | undefined;
  if (body.promoCode) {
    const promo = await backend.validatePromo(body.promoCode, {
      currency: body.currency, locationId: body.locationId,
      subtotal: pricing.value.subtotal,
    });
    if (!promo.valid) return err(400, 'PROMO_INVALID', { code: 'PROMO_INVALID' });
    discount = promo.discountAmount; promoId = promo.id;
  }

  // 3. Reject past slots before touching stock.
  const past = body.cart.slots.some((s) => Date.parse(s.dateTimeFrom) < Date.now());
  if (past) return err(400, 'booking_time_in_past', { code: 'BOOKING_TIME_IN_PAST' });

  // 4. Reserve stock, then create the booking transactionally. On ANY failure
  //    after locks exist, release them. The earlier implementation leaked
  //    reserved stock for its full TTL on every capacity conflict.
  const locks = await backend.lockStock({ ...body.cart, locationId: body.locationId,
                                          sessionId: body.idempotencyKey, source: 'kiosk' });
  if (!locks.ok) return err(409, 'INSUFFICIENT_STOCK', locks.detail);

  let created;
  try {
    created = await backend.createBookingTransactionally({
      idempotencyKey: body.idempotencyKey,
      locationId: body.locationId, serviceType: body.serviceType,
      windows: body.cart.slots, pricing: { ...pricing.value, discount, promoId },
      contact: { fullName: body.fullName, email: body.email ?? '', phone: body.phone ?? '' },
      participants: body.participants, parentBookingId: body.parentBookingId,
      paymentMethod: body.paymentMethod, stockLockIds: locks.ids, source: 'kiosk',
    });
  } catch (e) {
    await backend.releaseStockLocks(locks.ids);           // ← the fix
    if (isCapacityError(e)) return err(409, e.reason, { code: e.code });
    console.error('kiosk create failed', e);
    return err(500, 'booking_failed');                    // generic — no e.message
  }

  if (body.paymentMethod === 'counter') {
    await backend.confirmStockLocks(locks.ids, created.bookingId);
    if (promoId) await backend.recordPromoUse(promoId).catch(() => {});
    backend.sendConfirmation(created, body.locale).catch(() => {});  // fire-and-forget, but caught
    return ok({ bookingId: created.bookingId, shortId: created.shortId }, 201);
  }

  // Online: booking stays PENDING; the payment webhook confirms stock.
  const checkout = await backend.createCheckoutSession({
    amount: pricing.value.grandTotal - discount, currency: body.currency,
    bookingId: created.bookingId, shortId: created.shortId, locale: body.locale,
  });
  if (promoId) await backend.recordPromoUse(promoId).catch(() => {});
  return ok({ bookingId: created.bookingId, shortId: created.shortId,
              checkoutUrl: checkout.url }, 201);
}
```

Notes on that ordering:

- `recordPromoUse` runs **after** the booking is committed and its failure is
  swallowed: the earlier implementation let a "usage limit reached" throw escape *after*
  creating the booking, returning a 500 for a booking that existed.
- The confirmation email is fire-and-forget by design (a kiosk user is
  standing there; don't make them wait on SendGrid) — but with `.catch`, or
  every mail outage becomes an unhandled rejection.
- The checkout redirect base URL must be a required env var. The earlier implementation
  defaulted it to `http://localhost:3000`, so a missing env produced Stripe
  sessions whose success URLs pointed at localhost.

## Lookup

`GET /api/kiosk/booking/lookup?shortId=` — min 3 chars, trimmed, uppercased.

Three protections, all missing in the earlier implementation, all mandatory because the short
code is guessable (8 chars of a 31-char alphabet) and the endpoint is
reachable by anything that can reach the kiosk's origin:

1. **Rate limit** per IP (a small fixed-window counter is enough; the
   earlier implementation had a rate-limit helper used by exactly one other route).
2. **Mask the PII.** The edit flow needs to *show* whose booking it is, not
   exfiltrate it: return `fullName` as given (the person typed the code from
   their own confirmation), but mask email (`m•••@d•••.com`) and phone (last
   3 digits). The full values never leave the server on this route; the edit
   endpoint accepts new values without echoing old ones.
3. Return the same `notFound` for "no such booking" and "wrong location" —
   don't oracle which codes exist elsewhere.

Response data: the `LookupBooking` shape from
[state-machine.md](state-machine.md), with masked contact.

## Add-items and edit

Both operate on `bookingDocId` returned by lookup, and both need the guards
the earlier implementation was missing on one or the other:

| Guard | Why |
|---|---|
| Same `locationId` as the booking | cross-location tampering (the earlier implementation had this) |
| Booking not `CANCELLED` | the earlier implementation's add-items path locked stock and charged against cancelled bookings |
| Module/feature gates identical to create | the earlier implementation's edit route skipped them |
| Pricing recomputed **server-side, preserving the original discount** | the earlier implementation recalculated without the promo, silently un-discounting the booking on every edit |
| Mutations to `pricing` inside a transaction | two concurrent add-items requests lost one increment in the earlier implementation (plain read-modify-write) |
| Consumable lines appended as an *array push with a line id*, never a set-union | the earlier implementation used a Firestore `arrayUnion` of plain objects: buying the same item twice deduplicated to one line while the total still charged twice |
| Changing slots goes through the same transactional capacity re-check as create | the earlier implementation's edit wrote a new cart directly, able to double-book a station |

Add-items with `paymentMethod: 'online'` returns a `checkoutUrl` for the
delta amount only; the webhook must apply the delta by re-reading the booking
inside a transaction, not by writing back absolute totals stashed in payment
metadata (the earlier implementation did the latter, clobbering any concurrent change).

## Client fetch rule

Every kiosk request goes through **one** fetch wrapper that (a) attaches the
device key and (b) applies the offline base-URL failover
([realtime-offline.md](realtime-offline.md)). The earlier implementation had two components
calling bare `fetch` — exactly those two features (promo validation, booking
lookup) broke whenever the kiosk was in offline failover while everything
else kept working.
