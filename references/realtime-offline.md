# Realtime freshness and offline behavior

A kiosk shows availability that other channels are consuming in real time,
and it lives on venue hardware where the internet dies. Both problems have
cheap, proven solutions; neither needs websockets of your own or a state
library.

## The freshness signal

One number per location — `lastBookingChangeAt`, bumped inside every booking
write transaction ([booking-backend.md](booking-backend.md)) — pushed to
clients over whatever realtime channel the host already has (Firestore
`onSnapshot` in the earlier implementation; Supabase Realtime, an SSE endpoint, or 15 s
polling of a tiny JSON route all satisfy the same interface):

```ts
/** Resolve to an unsubscribe fn; invoke cb with a monotonically increasing
 *  signal (epoch millis) whenever bookings changed at this location. */
export type SubscribeFreshness =
  (locationId: string, cb: (signal: number) => void) => () => void;
```

Consumers react by **silently refetching** day + month availability — no
loading spinners, no error states for a background refresh — and dispatching
`REFRESH_SLOT` (never `SET_SLOT` — see [state-machine.md](state-machine.md))
with the fresh slot.

Share one subscription per location across all consumers with a refcounted
module-level registry, the pattern from the earlier implementation, generalized:

```ts
import { useEffect, useState } from 'react';

interface Sub { unsubscribe: () => void; listeners: Set<(n: number) => void>; last: number | null }
const subs = new Map<string, Sub>();

export function makeFreshnessHook(subscribe: SubscribeFreshness) {
  return function useBookingFreshness(locationId: string | null): number | null {
    const [signal, setSignal] = useState<number | null>(null);
    useEffect(() => {
      if (!locationId) return;
      let sub = subs.get(locationId);
      if (!sub) {
        const s: Sub = { unsubscribe: () => {}, listeners: new Set(), last: null };
        s.unsubscribe = subscribe(locationId, (n) => {
          s.last = n;
          s.listeners.forEach((l) => l(n));
        });
        subs.set(locationId, s);
        sub = s;
      }
      sub.listeners.add(setSignal);
      if (sub.last !== null) setSignal(sub.last);   // late subscriber catch-up
      return () => {
        const s = subs.get(locationId);
        if (!s) return;
        s.listeners.delete(setSignal);
        if (s.listeners.size === 0) { s.unsubscribe(); subs.delete(locationId); }
      };
    }, [locationId]);
    return signal;
  };
}
```

Consumers must skip the **first** signal (it arrives on subscribe and would
double the initial fetch) and skip repeats of the same value.

## Availability fetching

Month view and day view are separate fetches against the shared slots
endpoint. Rules that came from production behavior:

- **Cache at the CDN, not the client**: `Cache-Control: public, s-maxage=60,
  stale-while-revalidate=300` on the slots route keeps a bank of kiosks and
  the website from hammering the backend, while the freshness signal punches
  through staleness the moment anything changes (server-side, revalidate the
  slots cache tag inside the booking write path — *after* commit).
- **One automatic retry** on month-fetch failure, after 2 s — kiosk networks
  hiccup. Store the timer and clear it on unmount/param change; the earlier implementation's
  fire-and-forget retry outlived the component and set state after unmount.
- **Abort superseded requests.** Pricing, equipment and availability effects
  must carry an `AbortController` (or at minimum a request-id guard): rapid
  location/date changes otherwise let a slow early response overwrite a newer
  one. The earlier implementation had no cancellation anywhere.
- **Never swallow errors into `catch(() => {})`.** The earlier implementation did this in the
  pricing, equipment and locations fetches; the visible symptom was a submit
  button that greyed out forever with no explanation. Every fetch failure
  either renders a message or disables something *with* a message.

Stamp `lastUpdatedAt = Date.now()` on each successful day fetch and feed it
to the live indicator:

```
● live · 12s ago        ← pulsing dot; age from a 1 s now-tick
```

The indicator is trust UI — it tells the customer (and staff walking past)
that the station grid is current. Show it on time-select and participants, the
two screens where a stale grid causes a 409 at submit.

## Offline failover

The kiosk's job during an internet outage is to **keep selling for counter
payment**. The architecture (proven in the earlier implementation's ecosystem, and the
subject of the sibling `island-mode-server` skill — use it if the venue needs
the on-prem server itself):

```
kiosk ── /api/health poll (5 s, 3 s timeout) ──► cloud
  │ 3 consecutive failures → isOffline = true
  ├─► all API calls re-based onto the LAN fallback server URL
  ├─► yellow offline banner; online payment hidden; method forced to counter
  └─► recovery: same poll succeeds → base URL back to '' → banner gone
```

Client-side contract:

```ts
interface NetworkStatus {
  isOffline: boolean;
  apiBaseUrl: string;   // '' online; 'https://<lan-host>' when failed over
}
```

- The health poll needs its own **AbortController timeout (3 s)** — a hung
  request must count as a failure, not block the loop.
- Debounce the flip: 3 consecutive failures before going offline, so one
  dropped packet doesn't flash the banner.
- LAN candidates are probed in order (configured URL → mDNS name → static
  fallback IP) and the last-known-good is cached.
- **Every** kiosk request goes through the one fetch wrapper that applies
  `apiBaseUrl` (and the device key — [api-contract.md](api-contract.md)).
  The two components that bypassed it in the earlier implementation were exactly the two features
  that broke in island mode.

The LAN server must implement the same routes with the same contracts —
including capacity checks and idempotency. The earlier implementation's LAN mirror silently
ignored promo codes, participants and location ids, did no capacity check,
and minted a different shortId format; offline bookings were second-class in
ways nobody had listed. If the fallback can't honor a field, reject it
loudly rather than dropping it.

If the venue has no LAN fallback server, the degraded mode is still defined:
`isOffline` hides online payment and shows the banner; counter bookings
queue nowhere, so the create call fails visibly and staff take names on
paper. Do not fake success offline.

## What stays out

- No client cache layers (SWR/React Query) for kiosk data — the freshness
  signal plus CDN caching already give the right staleness envelope, and a
  second cache is a second thing to invalidate.
- No optimistic booking UI. A kiosk must never show "confirmed" before the
  server said so; the customer walks away on that word.
