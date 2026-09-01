# Screens and kiosk-grade UI machinery

The kiosk is one route rendering one component per `KioskStep`
([state-machine.md](state-machine.md)). This file is the screen contract —
structure, states and interaction rules. Appearance (colors, radii, fonts)
comes from the host's design system; visual choices are described only where
the *intent* matters (a kiosk in a bright room wants high contrast and huge
targets, whatever the palette).

## Screen flow

```
service-select ──► date-select ──► time-select ──► participants ──► equipment ──► summary ──► confirmation
      │                                                                                          (auto-reset 30 s)
      └─"I have a booking"──► booking-lookup ──► booking-edit ──┬─ add consumables ─► confirmation
                                                                └─ add slot ─► (service-select, parentBookingId set)
```

Header on every screen except entry/confirmation: back button (dispatches
`GO_BACK`), venue logo center (tap = cancel-confirm modal on any mid-flow
step), language switcher right.

## Screen contract

| Step | Renders | Required states | Key rules |
|---|---|---|---|
| *(no `?locationId`)* | location grid | loading, **empty**, list | Auto-redirect when exactly one location. The earlier implementation had no empty state — ship one. |
| `service-select` | large cards per service type + "I have a booking" | — | Doubles as the attract screen; inactivity overlay suppressed here. |
| `date-select` | month calendar, Monday-first | loading, no-slots day, past-day disabled | Availability-aware: dead days disabled. "Today" must come from a ticking clock — the earlier implementation memoized it once, so a kiosk left on overnight disabled the wrong days until reload. |
| `time-select` | slot grid with `available/total` per slot | loading, empty (`noSlots`), past-slot disabled | Live indicator + last-updated age. Past cutoff needs a ≤1 s tick (see below). |
| `participants` | station tiles + per-station name list, on-screen keyboard | station taken (disabled), validation (first name required) | Tiles re-render from `selectedSlot.takenStations`; `REFRESH_SLOT` already dropped stolen selections, so local UI state must derive from context, not a one-shot `useState` init. |
| `equipment` | category tabs, item grid, consumable quantity steppers | empty category, item disabled when no compatible consumable in stock | Selecting equipment auto-adds its compatible consumable line (match on `compatibilityKey`). Skippable step. |
| `summary` | order recap, promo code, contact fields, payment choice, sticky submit bar | promo error, submit error inline, submitting | All text fields open the on-screen keyboard. Payment method resolved at submit time (below). |
| `confirmation` | huge `shortId`, per-method instructions, QR for online payment, countdown | — | Auto-reset 30 s. |
| `booking-lookup` | code entry (min 3 chars) via on-screen keyboard | searching, not-found | 404 and network failure both show `notFound` — a kiosk user cannot act on the difference. |
| `booking-edit` | booking recap + actions: add consumables, add slot | no consumables configured, submitting, error | "Add slot" stores `parentBookingId`, resets to `service-select`, and shows a persistent "editing booking #X" banner until done. |

Errors are always inline text next to the action that failed — never toasts.
A kiosk user is standing at arm's length looking at one thing; a toast in a
corner is invisible and cannot be dismissed by the next customer.

## Payment method resolution

Offline state and feature flags can change *after* the user picked a method.
Resolve at submit, never trust a stale selection:

```ts
const onlineDisabled = isOffline || !stripeEnabled;   // host's flag seam
const effectiveMethod: 'online' | 'counter' =
  onlineDisabled ? 'counter' : (paymentMethod ?? 'counter');
```

The earlier implementation initialized the method once in `useState` and hid the online tile
when offline — but a user who picked "online" and then lost connectivity still
submitted an online booking that could never be paid.

**Online payment on a kiosk means a QR code.** The kiosk itself has no card
reader and must not navigate away to a payment page it can never come back
from (main flow). Render `checkoutUrl` as a real QR (any QR lib — it is the
one dependency this module justifies adding) so the customer pays on their
phone; keep the auto-reset running so an abandoned QR clears. The earlier implementation
shipped a text placeholder reading `QR: <url>...` — online payment was a
dead end in production.

## On-screen keyboard

Kiosks run with the OS keyboard disabled; every text input must set
`inputMode="none"` (suppresses any native keyboard) **and** open this
component. The earlier implementation set `inputMode="none"` on the contact, promo and
booking-code fields without wiring the keyboard — those screens were
unusable on a real touchscreen. Rule: if a field is focusable, it opens the
keyboard, no exceptions.

Bottom-sheet with its own value buffer; commit on save, discard on cancel:

```tsx
'use client';

import { useState } from 'react';

export type KeyboardMode = 'text' | 'email' | 'code';

export interface KeyboardStrings {
  shift: string; space: string; backspace: string; save: string; cancel: string;
}

interface KioskKeyboardProps {
  initialValue: string;
  title: string;
  mode?: KeyboardMode;
  /** Locale-specific extra row, e.g. Polish 'ą ć ę ł ń ó ś ż ź'. Comes from
   *  the dictionary so each language ships its own — never hardcode one. */
  accentRow?: string[];
  strings: KeyboardStrings;
  maxLength?: number;
  onSave: (value: string) => void;
  onCancel: () => void;
}

const LETTER_ROWS = [
  ['q', 'w', 'e', 'r', 't', 'y', 'u', 'i', 'o', 'p'],
  ['a', 's', 'd', 'f', 'g', 'h', 'j', 'k', 'l'],
  ['z', 'x', 'c', 'v', 'b', 'n', 'm'],
];
const DIGIT_ROW = ['1', '2', '3', '4', '5', '6', '7', '8', '9', '0'];
const EMAIL_ROW = ['@', '.', '-', '_'];

export function KioskKeyboard({
  initialValue, title, mode = 'text', accentRow,
  strings, maxLength = 60, onSave, onCancel,
}: KioskKeyboardProps) {
  const [value, setValue] = useState(initialValue);
  // Shift starts on for an empty text field so names capitalize naturally.
  const [shift, setShift] = useState(mode === 'text' && initialValue === '');

  const rows: string[][] = [
    DIGIT_ROW,
    ...LETTER_ROWS,
    ...(mode === 'email' ? [EMAIL_ROW] : []),
    ...(mode === 'text' && accentRow?.length ? [accentRow] : []),
  ];

  const press = (key: string) => {
    if (value.length >= maxLength) return;
    setValue(value + (shift ? key.toUpperCase() : key));
    if (shift) setShift(false);
  };

  return (
    // Layout intent: fixed bottom sheet, above everything, its own background.
    // touch-action: manipulation on the sheet AND every key kills the 300 ms
    // tap delay and double-tap zoom on kiosk browsers.
    <div
      role="dialog" aria-modal="true" aria-label={title}
      style={{ touchAction: 'manipulation' }}
    >
      <div>
        <span>{title}</span>
        <output aria-live="polite">{value || ' '}</output>
        <span>{value.length}/{maxLength}</span>
      </div>
      {rows.map((row, i) => (
        <div key={i}>
          {row.map((key) => (
            // Keys are min 44 px tall (56 px in the earlier implementation); see touch rules.
            <button key={key} type="button" onClick={() => press(key)}>
              {shift ? key.toUpperCase() : key}
            </button>
          ))}
        </div>
      ))}
      <div>
        {mode === 'text' && (
          <button type="button" aria-pressed={shift} onClick={() => setShift(!shift)}>
            {strings.shift}
          </button>
        )}
        <button type="button" onClick={() => value.length < maxLength && setValue(value + ' ')}>
          {strings.space}
        </button>
        <button type="button" aria-label={strings.backspace}
          onClick={() => setValue(value.slice(0, -1))}>
          ⌫
        </button>
        <button type="button" onClick={onCancel}>{strings.cancel}</button>
        <button type="button" onClick={() => onSave(value.trim())}>{strings.save}</button>
      </div>
    </div>
  );
}
```

Mode mapping: participant names and contact name → `text`; email → `email`;
phone → `code` is acceptable, digits dominate; promo and booking code →
`code` (auto-uppercase the result at the call site). The `mode`/`accentRow`
generalization is an **addition**: the earlier implementation shipped a single
QWERTY layout with one hardcoded diacritics row and used the keyboard on one
screen.

## Inactivity timer

```ts
import { useEffect, useRef } from 'react';

export function useInactivityTimer({
  timeoutMs = 120_000,
  onTimeout,
}: { timeoutMs?: number; onTimeout: () => void }) {
  const onTimeoutRef = useRef(onTimeout);
  // Ref updated in an effect, not in the render body: same stale-closure
  // protection, safe under StrictMode/concurrent double-renders.
  useEffect(() => { onTimeoutRef.current = onTimeout; }, [onTimeout]);

  useEffect(() => {
    let timer: ReturnType<typeof setTimeout>;
    const reset = () => {
      clearTimeout(timer);
      timer = setTimeout(() => onTimeoutRef.current(), timeoutMs);
    };
    const events = ['touchstart', 'mousedown', 'keydown', 'pointerdown'] as const;
    events.forEach((e) => window.addEventListener(e, reset, { passive: true }));
    reset();
    return () => {
      clearTimeout(timer);
      events.forEach((e) => window.removeEventListener(e, reset));
    };
  }, [timeoutMs]);
}
```

Wiring (values proven in production):

| Timer | Value | Behavior |
|---|---|---|
| Inactivity | 120 s | Show full-screen dim overlay ("touch to continue"); any tap hides it. On top of that, `RESET` the session — except on `service-select` (nothing to lose) and the `booking-*` side-flow steps (a staff-assisted flow shouldn't self-destruct mid-help). |
| Confirmation auto-reset | 30 s | Visible countdown ("returning in {n}s"), then `RESET`. Restart the countdown state whenever the effect re-runs — the earlier implementation kept a stale `remaining` across re-runs. |
| Now-tick for time UI | 1 s where seconds are shown | The earlier implementation ticked every 60 s under a "{n}s ago" label and a slot-started cutoff: the age lurched in minute jumps and started slots stayed tappable up to 59 s. Tick at the granularity you display. |

The overlay itself: full-screen fixed layer, `role="button"`, high-contrast
"touch the screen" affordance, fade in/out. Suppress it on `service-select` —
the entry screen *is* the idle state.

## Touch hardening

On the kiosk root element — this is what makes a web page feel like an
appliance, and none of it is optional on real hardware:

```tsx
<div
  className="kiosk-root"
  style={{
    userSelect: 'none', WebkitUserSelect: 'none',
    WebkitTouchCallout: 'none',          // no long-press callout on iOS/WebKit
    overscrollBehavior: 'none',           // no pull-to-refresh, no rubber-band
    touchAction: 'manipulation',          // no double-tap zoom, no 300 ms delay
  }}
  onContextMenu={(e) => e.preventDefault()}
>
```

Interaction rules:

- **Touch targets ≥ 44 px** on every interactive element; primary actions
  56-64 px. The earlier implementation's most-used quantity steppers were
  32 px while a less used screen's were 44 px; audit yours once, consistently.
- Tap feedback on everything actionable: a ~0.95 scale-on-press (Framer
  Motion `whileTap` in the earlier implementation; CSS `:active` transform works identically).
- Scroll containers get hidden scrollbars but *remain* scrollable; wide
  content scrolls inside its own container.
- No hover states as the only affordance — there is no hover on a kiosk.
- Do **not** strip focus outlines globally. The earlier implementation did; keyboard/switch
  access needs them, and a kiosk with an attached keypad is a real deployment.

Fullscreen is not the web app's job: launch the browser in kiosk mode (see
[operations.md](operations.md)). Do not add `requestFullscreen` calls.

## Modals

The kiosk hand-rolls its one modal (cancel-confirm on logo tap) rather than
using a host dialog primitive — a defensible choice on a touch appliance, but
it must still be a dialog: `role="dialog"`, `aria-modal="true"`, initial focus
on the safe action ("continue booking"), and the destructive action ("yes,
cancel") visually distinct. The earlier implementation had none of these. If the host has a
dialog primitive, prefer it.

## Strings

Every string is a dictionary key; slices are passed down as props from the
server-loaded dictionary (no client i18n runtime needed — the kiosk is one
page). Key tree the screens consume:

```
kiosk.selectLocation { title, subtitle, empty }
kiosk.selectService  { title, <per-service> { title, description }, haveBooking }
kiosk.selectDate     { title, months[12], weekdays[7], noSlots }
kiosk.selectTime     { title, available, noSlots }
kiosk.participants   { title, subtitle, participantLabel, stationLabel,
                       stationOccupied, addPerson, continue, required,
                       keyboard { title, shift, space, backspace, save, cancel },
                       accentRow }
kiosk.equipment      { title, subtitle, allCategories, selected, continue, skip,
                       empty, consumableLabel, outOfStock }
kiosk.summary        { title, service, date, time, equipment, addOns, stations,
                       total, discount, contact, fullName, email, phone,
                       payOnline, payCounter, payOnlineDesc, payCounterDesc,
                       submit, required, promo { label, apply, invalid },
                       termsAccept, termsLink }
kiosk.bookingLookup  { title, subtitle, placeholder, search, searching, notFound }
kiosk.bookingEdit    { title, existingBooking, addConsumables, addSlot,
                       consumableLabel, addMore, back, confirm, empty }
kiosk.confirmation   { title, orderNumber, payOnlineInstructions,
                       payCounterInstructions, newBooking, autoReset }
kiosk.idle           (string — "touch to continue")
kiosk.cancelConfirm  { title, yes, no }
kiosk.editingBooking (template with {shortId})
kiosk.live           { label, ago }   — ago is a template with {seconds}
kiosk.offline        (string — offline banner)
kiosk.back           (string)
```

Rules learned the hard way: make every key the screens read **required** in
the dictionary type (the earlier implementation's optional keys hid missing translations);
booking-availability error strings go through the dictionary too (the earlier implementation
hardcoded them in one language inside a shared hook); and the default station
label sent to the API is data, not UI copy — never fall back to a hardcoded
localized string for a value that gets persisted.
