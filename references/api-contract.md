# Kiosk API contract

Five kiosk-owned endpoints plus the shared read endpoints the screens consume.
The kiosk client is unauthenticated by nature, anyone standing at the screen
is a legitimate user, so the server, not the client, is where every trust
decision lives.

## Route surface

| Route | Method | Purpose | Auth |
|---|---|---|---|
| `/api/kiosk/booking/create` | POST | New booking, counter or online payment | device key |
| `/api/kiosk/booking/lookup?shortId=` | GET | Find booking by short code (edit flow) | device key + rate limit |
| `/api/kiosk/booking/add-items` | POST | Append consumables to an existing booking | device key |
| `/api/kiosk/booking/edit` | POST | Update participants / contact on a booking | device key |
| `/api/kiosk/promo/validate` | POST | Check a promo code against the cart, priced on the server | device key |
| `/api/pricing/get?locationId=` | GET | Price catalog (slots + consumables) | public read |
| `/api/inventory/equipment?locationId=&from=&to=` | GET | Equipment available in a window | public read |
| `/api/booking/slots?...` | GET | Month + day availability | public read, CDN-cached |

## Device auth

One shared header, `x-kiosk-api-key`, checked like this:

```ts
// file: lib/kiosk/server/auth.ts
import { createHash, timingSafeEqual } from 'node:crypto';

const digest = (s: string): Buffer => createHash('sha256').update(s).digest();

export function checkKioskKey(req: Request): boolean {
  const configured = process.env.KIOSK_API_KEY;
  if (!configured) return true;               // explicit opt-out: LAN-only / demo
  const sent = req.headers.get('x-kiosk-api-key');
  if (!sent) return false;                    // presence first: configured key = required header
  // Equal-length digests: a constant-time compare that leaks neither the key nor its length.
  return timingSafeEqual(digest(sent), digest(configured));
}
```

**When the key is configured, a missing header is a 401.** The earlier implementation checked
`if (sent && configured && sent !== configured)`, i.e. only a *wrong* key was
rejected and omitting the header bypassed auth entirely, on all four routes,
while the kiosk client never sent the header at all. Configure the key, send
it from the one `kioskFetch` wrapper ([Client fetch rule](#client-fetch-rule)),
server-injected, never `NEXT_PUBLIC_`, and treat the no-key mode as a conscious
deployment decision, not a fallback.

A shared device key is still a shared secret: it identifies "a kiosk", not
*which* kiosk, and cannot be revoked per-device. Good enough for one venue's
LAN; if kiosks live on the public internet, issue per-device tokens
(documented extension, not shipped; the sibling display module did
per-device pairing tokens and it is the right upgrade path).

## Error envelope

Every response, success or failure, uses one shape. The earlier implementation
mixed two and the client's error branch showed blank messages for one of them:

```ts
// file: lib/kiosk/envelope.ts
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
`STRIPE_DISABLED`, `PAYMENT_REQUIRED` (402, deployment/module blocked). Each
is a key under `kiosk.errors` ([screens.md](screens.md)); any other failure
shows `kiosk.errors.generic`.

## Create

Request:

```ts
// file: lib/kiosk/server/validation.ts
export interface KioskCreateRequest {
  idempotencyKey: string;            // the client sessionId - see below
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
  per consumable and the server fed them straight into the total, and into
  the Stripe charge amount. A tampered request could buy anything for zero.
  The server resolves every price from its own catalog by id
  ([booking-backend.md](booking-backend.md)); the client's displayed total is
  a preview, and a mismatch between preview and charge is a bug to surface,
  not reconcile silently.
- **No redundant `slotIds` array.** The earlier implementation sent N copies of the same
  slotId alongside the cart and the server never read them.

Response `201`: `{ ok: true, data: { bookingId, shortId, checkoutUrl? } }`.
`checkoutUrl` present iff `paymentMethod === 'online'`, render it as a QR
([screens.md](screens.md)).

### Validation

Dependency-free guards (swap for the host's schema library if it has one:
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
function isOptionalString(v: unknown, max: number): boolean {
  return v === undefined || (typeof v === 'string' && v.length <= max);
}

/** The cart as create and the promo check both take it: slots required,
 *  equipment optional, consumables normalised to an array. */
export function parseCart(value: unknown): KioskCreateRequest['cart'] | { error: string } {
  const cart = typeof value === 'object' && value !== null
    ? value as Record<string, unknown> : undefined;
  const slots = Array.isArray(cart?.slots) ? cart.slots : null;
  if (!slots || slots.length === 0 || slots.length > MAX_SLOTS)
    return { error: 'slots_invalid' };
  for (const s of slots as Array<Record<string, unknown>>) {
    if (typeof s !== 'object' || s === null ||
        !isNonEmptyString(s.slotId, 100) || !isNonEmptyString(s.station, 100) ||
        !isIsoDate(s.dateTimeFrom) || !isIsoDate(s.dateTimeTo))
      return { error: 'slot_shape_invalid' };
  }
  const equipment = cart?.equipment;
  if (equipment !== undefined) {
    if (!Array.isArray(equipment) || equipment.length > MAX_LINES)
      return { error: 'equipment_invalid' };
    for (const e of equipment as Array<Record<string, unknown>>) {
      if (typeof e !== 'object' || e === null || !isNonEmptyString(e.slug, 100) ||
          !isOptionalString(e.inventoryItemId, 100) ||
          (e.quantity !== undefined && !isQuantity(e.quantity)))
        return { error: 'equipment_shape_invalid' };
    }
  }
  const consumables = cart?.consumables === undefined ? [] : cart.consumables;
  if (!Array.isArray(consumables) || consumables.length > MAX_LINES)
    return { error: 'consumables_invalid' };
  for (const c of consumables as Array<Record<string, unknown>>) {
    if (typeof c !== 'object' || c === null ||
        !isNonEmptyString(c.itemId, 100) || !isQuantity(c.quantity))
      return { error: 'consumable_shape_invalid' };
  }
  return { ...(cart as KioskCreateRequest['cart']), consumables } as KioskCreateRequest['cart'];
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
  if (!isNonEmptyString(b.locale, 20)) return { error: 'locale_invalid' };
  // Optional fields reach the backend as they are, so each one is either absent
  // or the type the interface says: a number or an object is not a promo code.
  if (!isOptionalString(b.email, 200) || !isOptionalString(b.phone, 40) ||
      !isOptionalString(b.promoCode, 50) || !isOptionalString(b.parentBookingId, 100))
    return { error: 'optional_field_invalid' };

  const cart = parseCart(b.cart);
  if ('error' in cart) return cart;
  const participants = b.participants === undefined ? [] : b.participants;
  if (!Array.isArray(participants) || participants.length > MAX_PARTICIPANTS ||
      participants.some((p) => !isNonEmptyString(p, NAME_MAX)))
    return { error: 'participants_invalid' };

  // Shape now proven field-by-field; the cart is the normalised one.
  return { ...(body as KioskCreateRequest), cart };
}
```

Bounds matter as much as shapes: quantities are positive integers with a cap
(a negative quantity in the earlier implementation underflowed the total into a discount),
arrays are capped, and every string has a max length.

### Idempotency

The client sends its `sessionId` as `idempotencyKey`. Server-side:

```
1. Look up an existing booking where idempotencyKey == key (indexed field).
2. Found and < 10 min old -> return it (200, same payload shape) - this is a
   retry of a request whose response was lost.
3. Not found -> create, storing the key on the booking.
```

The earlier implementation's only duplicate protection was a client-side `submitting` boolean:
a tap racing a re-render, a network retry, or a flaky kiosk touch controller
produced double bookings. The server minted its own key from `Date.now()`,
which two terminals can share in the same millisecond.

### Route skeleton

The full order of operations, with backend calls behind the `KioskBackend`
seam ([booking-backend.md](booking-backend.md)). `backend` is the host's
implementation of that interface, exported from `lib/kiosk/server/instance.ts`;
the `@/` alias is a default Next.js app's, so use the host's own if it differs.

```ts
// file: app/api/kiosk/booking/create/route.ts
import { err, ok } from '@/lib/kiosk/envelope';
import { checkKioskKey } from '@/lib/kiosk/server/auth';
import { isCapacityError } from '@/lib/kiosk/server/backend';
import { backend } from '@/lib/kiosk/server/instance';
import { parseCreateRequest } from '@/lib/kiosk/server/validation';

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

  // 1. Server-side pricing - the only prices that exist.
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
    await backend.releaseStockLocks(locks.ids);           // <- the fix
    if (isCapacityError(e)) return err(409, e.reason, { code: e.code });
    console.error('kiosk create failed', e);
    return err(500, 'booking_failed');                    // generic - no e.message
  }

  if (body.paymentMethod === 'counter') {
    // The booking exists from here on, so it is answered as created. A confirm
    // that failed after its own retry has flagged the booking for operators.
    await backend.confirmStockLocks(locks.ids, created.bookingId)
      .catch((e) => console.error('kiosk stock confirm failed', created.bookingId, e));
    if (promoId) await backend.recordPromoUse(promoId).catch(() => {});
    backend.sendConfirmation(created, body.locale).catch(() => {});  // fire-and-forget, but caught
    return ok({ bookingId: created.bookingId, shortId: created.shortId }, 201);
  }

  // Online: booking stays PENDING; the payment webhook confirms stock. A
  // checkout that cannot be created is a failure path after the locks exist:
  // fail the booking, which releases its locks and stations. If that fails too,
  // the TTL sweep and the stale-PENDING job are the net.
  let checkout: { url: string };
  try {
    checkout = await backend.createCheckoutSession({
      amount: pricing.value.grandTotal - discount, currency: body.currency,
      bookingId: created.bookingId, shortId: created.shortId, locale: body.locale,
    });
  } catch (e) {
    await backend.failPendingBooking(created.bookingId).catch(() => {});
    console.error('kiosk checkout failed', e);
    return err(502, 'checkout_failed');
  }
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
  standing there; don't make them wait on SendGrid), but with `.catch`, or
  every mail outage becomes an unhandled rejection.
- The checkout redirect base URL, `KIOSK_PUBLIC_BASE_URL`, is required whenever
  online payment is on. The earlier implementation defaulted it to
  `http://localhost:3000`, so a missing env produced Stripe sessions whose
  success URLs pointed at localhost.
- A checkout session that cannot be created fails the booking through
  `failPendingBooking` and answers 502; the summary shows the error inline and
  counter payment stays available. Left PENDING, the booking would hold its
  stations and its locks until the cleanup jobs ran.

### Promo check

The summary screen's promo field asks the server before submit, so an invalid
code shows `PROMO_INVALID` at the field rather than at create. The body carries
the cart, never an amount; the server prices it the way create does and writes
nothing. Create still re-validates: a check is a preview, not a reservation.

```ts
// file: app/api/kiosk/promo/validate/route.ts
import { err, ok } from '@/lib/kiosk/envelope';
import { checkKioskKey } from '@/lib/kiosk/server/auth';
import { backend } from '@/lib/kiosk/server/instance';
import { parseCart } from '@/lib/kiosk/server/validation';

export async function POST(request: Request) {
  if (!checkKioskKey(request)) return err(401, 'unauthorized');
  let raw: unknown;
  try { raw = await request.json(); } catch { return err(400, 'invalid_json'); }
  if (typeof raw !== 'object' || raw === null) return err(400, 'invalid_body');
  const b = raw as Record<string, unknown>;
  const { serviceType, locationId, currency, promoCode } = b;
  if (typeof serviceType !== 'string' || typeof locationId !== 'string' ||
      typeof currency !== 'string' || typeof promoCode !== 'string' ||
      promoCode.length === 0 || promoCode.length > 50)
    return err(400, 'invalid_body');
  const cart = parseCart(b.cart);
  if ('error' in cart) return err(400, cart.error);

  const pricing = await backend.resolvePricing({
    locationId, serviceType, currency, slots: cart.slots, consumables: cart.consumables,
  });
  if (!pricing.ok) return err(400, pricing.error);
  const promo = await backend.validatePromo(promoCode, {
    currency, locationId, subtotal: pricing.value.subtotal,
  });
  if (!promo.valid) return err(400, 'PROMO_INVALID', { code: 'PROMO_INVALID' });
  return ok({ discount: promo.discountAmount,
              grandTotal: pricing.value.grandTotal - promo.discountAmount });
}
```

## Lookup

`GET /api/kiosk/booking/lookup?shortId=`, min 3 chars, trimmed, uppercased.

Three protections, all missing in the earlier implementation, all mandatory because the short
code is guessable (8 chars of a 31-char alphabet) and the endpoint is
reachable by anything that can reach the kiosk's origin:

1. **Rate limit** per IP (a small fixed-window counter is enough; the
   earlier implementation had a rate-limit helper used by exactly one other route).
2. **Mask the PII.** The edit flow needs to *show* whose booking it is, not
   exfiltrate it: return `fullName` as given (the person typed the code from
   their own confirmation), but mask email (`m***@d***.com`) and phone (last
   3 digits). The full values never leave the server on this route; the edit
   endpoint accepts new values without echoing old ones.
3. Return the same `notFound` for "no such booking" and "wrong location";
   don't oracle which codes exist elsewhere.

Response data: the `LookupBooking` shape from
[state-machine.md](state-machine.md), with masked contact.

## Add-items and edit

Both operate on `bookingDocId` returned by lookup, and both need the guards
the earlier implementation was missing on one or the other:

| Guard | Why |
|---|---|
| Same `locationId` as the booking | cross-location tampering (the earlier implementation had this) |
| An `idempotencyKey` per attempt, minted when the add dialog opens and checked like create's | a double tap or a retried request appended the same lines twice; a disabled button is not a server guard |
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
device key, (b) applies the offline base-URL failover
([realtime-offline.md](realtime-offline.md)) and (c) unwraps the error
envelope. The earlier implementation had two components calling bare `fetch`,
exactly those two features (promo validation, booking lookup) broke whenever the
kiosk was in offline failover while everything else kept working.

```ts
// file: lib/kiosk/kiosk-fetch.ts
import type { ApiErr, ApiOk } from './envelope';

/** A failed envelope or an unreachable server. `code` is what a screen maps to
 *  a dictionary key; `detail` carries item/requested/available. */
export class KioskApiError extends Error {
  constructor(
    public status: number,
    public error: string,
    public code?: string,
    public detail: Omit<ApiErr, 'ok' | 'error' | 'code'> = {},
  ) { super(error); }
}

export interface KioskFetchConfig {
  /** KIOSK_API_KEY, read on the server and passed down as a prop; null when unset. */
  deviceKey: string | null;
  /** NetworkStatus.apiBaseUrl at call time: '' online, the LAN URL when failed over. */
  getApiBaseUrl: () => string;
}

export type KioskFetch = <T>(path: string, init?: RequestInit) => Promise<T>;

export function createKioskFetch({ deviceKey, getApiBaseUrl }: KioskFetchConfig): KioskFetch {
  return async function kioskFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
    const headers = new Headers(init.headers);
    if (deviceKey) headers.set('x-kiosk-api-key', deviceKey);
    if (init.body != null && !headers.has('content-type')) {
      headers.set('content-type', 'application/json');
    }
    let res: Response;
    try {
      res = await fetch(`${getApiBaseUrl()}${path}`, { ...init, headers, cache: 'no-store' });
    } catch (e) {
      if (e instanceof DOMException && e.name === 'AbortError') throw e;  // superseded, not failed
      throw new KioskApiError(0, 'network_error');
    }
    const body = (await res.json().catch(() => null)) as ApiOk<T> | ApiErr | null;
    if (!body || typeof body !== 'object' || !('ok' in body)) {
      throw new KioskApiError(res.status, 'invalid_response');
    }
    if (!body.ok) {
      const { ok: _ok, error, code, ...detail } = body;
      throw new KioskApiError(res.status, error, code, detail);
    }
    return body.data;
  };
}
```

The kiosk page is a server component: it reads `KIOSK_API_KEY` and
`KIOSK_LAN_FALLBACK_URL` from `process.env` and passes them to the client
provider as props, which builds the one `kioskFetch` with `createKioskFetch`
and hands it to every hook and screen. Nothing else in the kiosk calls `fetch`,
with one exception: the health poll is not a kiosk request but the probe that
decides the base URL, so it always asks the cloud origin with plain `fetch` and
its own 3 s abort.
