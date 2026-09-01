# Operating a kiosk fleet

The code is half the product; the other half is a physical machine in a lobby
that nobody looks at until it misbehaves. This file covers deployment,
access gating, and the operator surface, including the gaps the earlier
implementation shipped with, marked as extensions where this skill does not
carry code.

## Launching the kiosk

Fullscreen is the OS/browser's job, not the web app's (no `requestFullscreen`
calls — they need a user gesture and break on reload). Proven setups:

**Windows + Edge kiosk mode** (what the earlier implementation deployed):

```powershell
msedge.exe --kiosk "https://<host>/<locale>/kiosk?locationId=<id>" `
  --edge-kiosk-type=fullscreen --no-first-run
```

Pair with Windows *Assigned Access* so the shell itself is locked to the
browser, and an auto-restart scheduled task for the morning. Chrome:
`--kiosk`; ChromeOS: managed kiosk app.

Hard rules for the URL: it **must carry `?locationId=`** — a kiosk that boots
to the location selector after a power cut is a support call. And if the app
is a PWA, check `manifest.json`'s `start_url`: in the earlier implementation
it pointed at the website root, so "installing" the kiosk page installed the
wrong app.

Machine checklist: disable OS sleep and screen blanking; disable Windows
touch gestures (edge swipes); auto-login; browser set to clear nothing on
restart (session state is in-memory anyway); NTP on — slot cutoffs compare
against the device clock.

## Access gating

Two distinct concerns the earlier implementation conflated under one route
group:

1. **Deployment gating** — on a multi-tenant/demo deployment, the kiosk
   route can require a privileged staff login before rendering at all (the
   earlier implementation locked its `main` deployment behind a superadmin
   session while customer forks were public). This is a host seam: reuse the
   host's staff auth, render its login form in a kiosk-styled shell.
2. **Feature gating** — kiosk on/off per deployment, plus a billing
   "module blocked" state. Gate **both** the page (server layout) and
   **every API route**; the earlier implementation gated the page but left
   the API live with the feature off, and its edit route skipped the block
   check its siblings had. One helper, called everywhere, or the gates drift.

There is deliberately no per-customer auth: anyone at the screen may book.
That is why the server trusts nothing from the client
([api-contract.md](api-contract.md)).

## The operator surface

What staff and admins must be able to see and do. Items marked *(extension)*
were absent in the earlier implementation; they are documented so you ship or
consciously skip them, not forget them.

| Need | Mechanism |
|---|---|
| Is the kiosk alive? | *(extension)* Heartbeat: the kiosk pings a status endpoint every ~60 s with a device id; admin list shows last-seen. The earlier implementation had zero remote visibility — a crashed kiosk was discovered by walking past. The sibling display module's pairing + last-seen model is the pattern to copy. |
| Which bookings came from kiosks? | `source: 'kiosk'` on every booking; filterable in the staff booking list. |
| Take counter payment | Staff panel: look up by `shortId`, mark paid. This is the kiosk's main daily touchpoint with staff — it must be fast. |
| Stock that failed to confirm | *(extension)* A booking flag (`stockConfirmFailed`) surfaced in the staff panel — see [booking-backend.md](booking-backend.md); the earlier implementation logged to console and told no one. |
| Paid-but-lost-capacity refunds | `payment.requiresRefund` flag surfaced as a staff task queue; never auto-refund. |
| Inventory audit trail | Confirm/release writes an `inventoryLogs`-style entry: who, what, when, which booking. |
| Kill switch | The feature gate doubles as one: turning the kiosk feature off must blank the screen (the layout gate) *and* the API. |
| Abandoned-session hygiene | In-memory state + 120 s inactivity reset + 30 s confirmation reset. Verify on hardware: the previous customer's name must never be on screen for the next one. |

## Environment

| Var | Required | Notes |
|---|---|---|
| `KIOSK_API_KEY` | strongly recommended | Missing = open API, a logged deployment decision ([api-contract.md](api-contract.md)) |
| Public base URL | **yes** for online payment | No localhost default — the earlier implementation generated payment redirects to `http://localhost:3000` when unset |
| Payment provider keys + webhook secret | for online payment | Counter-only kiosks run without them |
| LAN fallback URL | if island mode | See [realtime-offline.md](realtime-offline.md) |

## Smoke test after any deploy

1. Boot URL lands on `service-select` with the right location and language.
2. Full happy path to a counter booking; shortId visible; staff panel shows it.
3. Online path renders a scannable QR; paying on a phone confirms the booking.
4. Book the same slot's last station from another device mid-flow → kiosk
   drops the station from the grid without navigating away.
5. Pull the network cable: banner appears ≤20 s, online payment gone,
   counter booking still completes (if LAN server) or fails visibly (if not).
6. Walk away at the summary screen: overlay at 120 s, reset after; no
   personal data on screen.
