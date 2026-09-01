# The kiosk state machine

One React context holding a pure reducer. Every screen transition is a reducer
action, not router navigation — the kiosk is a single URL and the browser back
button must never be part of the flow. The reducer is the only piece of the
module complex enough to deserve tests, and it has them (end of this file).

## Canonical vocabulary

| Canonical | What it means | Rename in your host to |
|---|---|---|
| `serviceType` | the bookable activity variant | court type, room, activity… |
| `station` | one bookable unit of capacity within a slot | bay, court, seat, table |
| `AvailabilitySlot` | time slot with per-station availability | same concept |
| `equipment` | time-bound rentable items that come back | rackets, karts, gear |
| `consumable` | stock-decremented items that do not | balls, tokens, credits |
| `compatibilityKey` | links a consumable to the equipment it fits | size, gauge, type |
| `participants` | people occupying a station | players, guests |

Do the rename once, before generating, and apply it everywhere — types, routes,
component names, strings.

## State shape

```ts
export interface AvailabilitySlot {
  slotId: string;
  time: string;                  // 'HH:mm' display label
  dateTimeFrom: string;          // ISO, venue timezone resolved server-side
  dateTimeTo: string;
  available: number;             // stations still free in this slot
  takenStations: string[];       // station labels already booked
  price?: number;                // minor units; display only — server re-prices
}

export interface EquipmentSelection {
  slug: string;
  name: string;
  compatibilityKey: string;      // links to consumables
  category: string;
  image: string;
}

export interface CatalogItem {
  id: string;
  name: string;
  compatibilityKey?: string;
  unitLabel?: string;            // e.g. "50 pcs" per unit
  prices: Partial<Record<string, number>>;  // currency -> minor units
}

export interface CartLine {
  item: CatalogItem;
  quantity: number;
}

export interface ContactInfo {
  fullName: string;
  email: string;
  phone: string;
}

export interface BookingResult {
  bookingId: string;
  shortId: string;               // the human code shown on the confirmation screen
  checkoutUrl?: string;          // present when paymentMethod === 'online'
}

export interface LookupBooking {
  bookingDocId: string;
  shortId: string;
  serviceType: string;
  date: string;                  // 'YYYY-MM-DD'
  time: string;
  equipment: Array<{ name: string; compatibilityKey: string; image?: string }>;
  consumables: Array<{
    itemId: string;
    name: string;
    compatibilityKey?: string;
    unitLabel?: string;
    price: number;
    quantity: number;
  }>;
  pricing: {
    slotsTotal: number;
    consumablesTotal: number;
    grandTotal: number;
    currency: string;
  };
  locationId: string;
  participants?: string[];
  contactInfo?: ContactInfo | null;
  status?: string;
}

export type KioskStep =
  | 'service-select'     // entry / attract screen — also hosts "I have a booking"
  | 'date-select'
  | 'time-select'
  | 'participants'       // pick stations, enter names via on-screen keyboard
  | 'equipment'
  | 'summary'
  | 'confirmation'
  | 'booking-lookup'     // side flow: find existing booking by short code
  | 'booking-edit';      // side flow: add consumables / add a slot to it

export interface KioskState {
  sessionId: string;             // idempotency key — sent on create, see api-contract.md
  step: KioskStep;
  serviceType: string | null;
  selectedDate: string | null;   // 'YYYY-MM-DD'
  selectedSlot: AvailabilitySlot | null;
  selectedStations: string[];
  participantNames: string[][];  // index-aligned with selectedStations
  selectedEquipment: EquipmentSelection[];
  cart: CartLine[];
  contactInfo: ContactInfo;
  contactEdited: boolean;        // once true, participant-name seeding stops overwriting
  paymentMethod: 'online' | 'counter' | null;
  bookingResult: BookingResult | null;
  lookupBooking: LookupBooking | null;
  parentBookingId: string | null; // set when adding a slot to an existing booking
}
```

## Actions

```ts
export type KioskAction =
  | { type: 'SET_SERVICE'; serviceType: string }
  | { type: 'SET_DATE'; date: string }
  | { type: 'SET_SLOT'; slot: AvailabilitySlot }        // user tap — navigates
  | { type: 'REFRESH_SLOT'; slot: AvailabilitySlot }    // background refresh — never navigates
  | { type: 'SET_PARTICIPANTS'; stations: string[]; participantNames: string[][] }
  | { type: 'TOGGLE_EQUIPMENT'; equipment: EquipmentSelection }
  | { type: 'SET_EQUIPMENT'; equipment: EquipmentSelection[] }
  | { type: 'SET_CART'; cart: CartLine[] }
  | { type: 'UPDATE_QUANTITY'; itemId: string; quantity: number }
  | { type: 'SET_CONTACT'; contactInfo: ContactInfo }
  | { type: 'SET_PAYMENT_METHOD'; method: 'online' | 'counter' }
  | { type: 'SET_BOOKING_RESULT'; result: BookingResult }
  | { type: 'SET_LOOKUP_BOOKING'; booking: LookupBooking }
  | { type: 'SET_PARENT_BOOKING_ID'; parentBookingId: string }
  | { type: 'GO_TO_STEP'; step: KioskStep }
  | { type: 'GO_BACK' }
  | { type: 'RESET' };
```

`SET_SLOT` vs `REFRESH_SLOT` is the load-bearing distinction. The kiosk
silently refetches availability whenever another terminal or the website books
something (see [realtime-offline.md](realtime-offline.md)). If that refresh
re-dispatches the same action a user tap uses, and that action navigates, a
background update **teleports the user back to station selection from the
summary screen**, discarding their progress. The earlier implementation had exactly this
bug. Data updates and navigation are separate actions; keep them separate.

## Reducer

```ts
const STEP_ORDER: KioskStep[] = [
  'service-select', 'date-select', 'time-select',
  'participants', 'equipment', 'summary', 'confirmation',
];

function generateSessionId(): string {
  // Doubles as the create-request idempotency key, so it must be unique per
  // flow attempt — not per millisecond. RESET regenerates it.
  return `kiosk_${crypto.randomUUID()}`;
}

export function createInitialState(): KioskState {
  return {
    sessionId: generateSessionId(),
    step: 'service-select',
    serviceType: null,
    selectedDate: null,
    selectedSlot: null,
    selectedStations: [],
    participantNames: [],
    selectedEquipment: [],
    cart: [],
    contactInfo: { fullName: '', email: '', phone: '' },
    contactEdited: false,
    paymentMethod: null,
    bookingResult: null,
    lookupBooking: null,
    parentBookingId: null,
  };
}

export function kioskReducer(state: KioskState, action: KioskAction): KioskState {
  switch (action.type) {
    case 'SET_SERVICE':
      // Clear everything downstream — selections made for one service type are
      // likely invalid for another. The empty cart is re-seeded by the pricing
      // hook when it sees cart.length === 0.
      return {
        ...state,
        serviceType: action.serviceType,
        step: 'date-select',
        selectedDate: null,
        selectedSlot: null,
        selectedStations: [],
        participantNames: [],
        selectedEquipment: [],
        cart: [],
      };
    case 'SET_DATE':
      return { ...state, selectedDate: action.date, step: 'time-select' };
    case 'SET_SLOT':
      return { ...state, selectedSlot: action.slot, step: 'participants' };
    case 'REFRESH_SLOT': {
      if (!state.selectedSlot || state.selectedSlot.slotId !== action.slot.slotId) {
        return state;
      }
      // Stations the user picked may have been taken by another channel while
      // they browsed. Drop them here so the submit payload can never contain a
      // station the server will reject — and keep participantNames aligned by
      // filtering both arrays on the same predicate.
      const takenNow = new Set(action.slot.takenStations);
      const keep = state.selectedStations.map((s) => !takenNow.has(s));
      return {
        ...state,
        selectedSlot: action.slot,
        selectedStations: state.selectedStations.filter((_, i) => keep[i]),
        participantNames: state.participantNames.filter((_, i) => keep[i]),
      };
    }
    case 'SET_PARTICIPANTS': {
      const first = action.participantNames[0]?.[0] ?? '';
      return {
        ...state,
        selectedStations: action.stations,
        participantNames: action.participantNames,
        // Seed the contact name from the first participant as a convenience,
        // but never overwrite a name the user typed on the summary screen.
        contactInfo: state.contactEdited || first === ''
          ? state.contactInfo
          : { ...state.contactInfo, fullName: first },
      };
    }
    case 'TOGGLE_EQUIPMENT': {
      const exists = state.selectedEquipment.some((e) => e.slug === action.equipment.slug);
      return {
        ...state,
        selectedEquipment: exists
          ? state.selectedEquipment.filter((e) => e.slug !== action.equipment.slug)
          : [...state.selectedEquipment, action.equipment],
      };
    }
    case 'SET_EQUIPMENT':
      return { ...state, selectedEquipment: action.equipment };
    case 'SET_CART':
      return { ...state, cart: action.cart };
    case 'UPDATE_QUANTITY':
      return {
        ...state,
        cart: state.cart.map((line) =>
          line.item.id === action.itemId
            ? { ...line, quantity: Math.max(0, action.quantity) }
            : line
        ),
      };
    case 'SET_CONTACT':
      return { ...state, contactInfo: action.contactInfo, contactEdited: true };
    case 'SET_PAYMENT_METHOD':
      return { ...state, paymentMethod: action.method };
    case 'SET_BOOKING_RESULT':
      return { ...state, bookingResult: action.result, step: 'confirmation' };
    case 'SET_LOOKUP_BOOKING':
      return { ...state, lookupBooking: action.booking, step: 'booking-edit' };
    case 'SET_PARENT_BOOKING_ID':
      return { ...state, parentBookingId: action.parentBookingId };
    case 'GO_TO_STEP':
      return { ...state, step: action.step };
    case 'GO_BACK': {
      // The side-flow steps are not in STEP_ORDER; without these branches a
      // back press there is a silent no-op (the earlier implementation's back button had to
      // fall back to a full RESET, losing the looked-up booking).
      if (state.step === 'booking-lookup') return { ...state, step: 'service-select' };
      if (state.step === 'booking-edit') {
        return { ...state, step: 'booking-lookup', lookupBooking: null };
      }
      const i = STEP_ORDER.indexOf(state.step);
      if (i <= 0) return state;
      return { ...state, step: STEP_ORDER[i - 1] ?? state.step };
    }
    case 'RESET':
      return createInitialState();
    default:
      return state;
  }
}
```

## Provider

```tsx
'use client';

import React, { createContext, useCallback, useContext, useReducer } from 'react';

interface KioskContextValue {
  state: KioskState;
  dispatch: React.Dispatch<KioskAction>;
  reset: () => void;
  goBack: () => void;
}

const KioskContext = createContext<KioskContextValue | undefined>(undefined);

export function KioskProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(kioskReducer, undefined, createInitialState);
  const reset = useCallback(() => dispatch({ type: 'RESET' }), []);
  const goBack = useCallback(() => dispatch({ type: 'GO_BACK' }), []);
  return (
    <KioskContext.Provider value={{ state, dispatch, reset, goBack }}>
      {children}
    </KioskContext.Provider>
  );
}

export function useKiosk(): KioskContextValue {
  const ctx = useContext(KioskContext);
  if (!ctx) throw new Error('useKiosk must be used within a KioskProvider');
  return ctx;
}
```

## Rules the orchestrator must follow

- **Location lives in the URL** (`?locationId=`), not in state. A location
  change must dispatch `RESET`: the cart holds `CatalogItem`s (with backend
  item ids) from the previous location's pricing, and the earlier implementation shipped a
  bug where switching locations submitted the old location's item ids.
- **No persistence.** Kiosk state is deliberately in-memory only: an abandoned
  session must not leak one customer's name and email to the next customer.
  Do not "improve" this with localStorage.
- **`sessionId` goes on the wire.** Send it as the idempotency key on booking
  create ([api-contract.md](api-contract.md)). The earlier implementation generated it and
  never read it, while the server minted timestamp keys that collide across
  terminals.
- **Session lifecycle timers** (values proven in production): inactivity
  timeout 120 s → dim overlay; any touch wakes; timeout on any step except the
  entry and side-flow steps also dispatches `RESET`. Confirmation screen
  auto-resets after 30 s with a visible countdown. See
  [screens.md](screens.md) for the components.

## Reducer tests

Run with the host's test runner; these are runner-agnostic apart from the
imports. They encode the invariants the hardening added — keep them green.

```ts
import { describe, expect, it } from 'vitest';
import {
  createInitialState, kioskReducer,
  type AvailabilitySlot, type KioskState,
} from './kiosk-context';

const slot = (over: Partial<AvailabilitySlot> = {}): AvailabilitySlot => ({
  slotId: 's1', time: '10:00',
  dateTimeFrom: '2026-09-01T10:00:00+02:00', dateTimeTo: '2026-09-01T11:00:00+02:00',
  available: 3, takenStations: [], ...over,
});

const atStep = (step: KioskState['step'], over: Partial<KioskState> = {}): KioskState =>
  ({ ...createInitialState(), step, ...over });

describe('kioskReducer', () => {
  it('walks the linear flow forward', () => {
    let s = createInitialState();
    s = kioskReducer(s, { type: 'SET_SERVICE', serviceType: 'classic' });
    expect(s.step).toBe('date-select');
    s = kioskReducer(s, { type: 'SET_DATE', date: '2026-09-01' });
    expect(s.step).toBe('time-select');
    s = kioskReducer(s, { type: 'SET_SLOT', slot: slot() });
    expect(s.step).toBe('participants');
  });

  it('SET_SERVICE clears downstream selections', () => {
    const dirty = atStep('summary', {
      selectedDate: '2026-09-01', selectedSlot: slot(),
      selectedStations: ['A'], participantNames: [['Kim']],
      cart: [{ item: { id: 'i1', name: 'x', prices: {} }, quantity: 2 }],
    });
    const s = kioskReducer(dirty, { type: 'SET_SERVICE', serviceType: 'ar' });
    expect(s.selectedSlot).toBeNull();
    expect(s.selectedStations).toEqual([]);
    expect(s.cart).toEqual([]);
  });

  it('REFRESH_SLOT never changes the step', () => {
    const s = atStep('summary', { selectedSlot: slot() });
    const out = kioskReducer(s, { type: 'REFRESH_SLOT', slot: slot({ available: 1 }) });
    expect(out.step).toBe('summary');
    expect(out.selectedSlot?.available).toBe(1);
  });

  it('REFRESH_SLOT drops stations taken elsewhere and keeps names aligned', () => {
    const s = atStep('summary', {
      selectedSlot: slot(),
      selectedStations: ['A', 'B', 'C'],
      participantNames: [['Ana'], ['Ben'], ['Cy']],
    });
    const out = kioskReducer(s, {
      type: 'REFRESH_SLOT', slot: slot({ takenStations: ['B'] }),
    });
    expect(out.selectedStations).toEqual(['A', 'C']);
    expect(out.participantNames).toEqual([['Ana'], ['Cy']]);
  });

  it('REFRESH_SLOT ignores a different slotId', () => {
    const s = atStep('summary', { selectedSlot: slot() });
    const out = kioskReducer(s, { type: 'REFRESH_SLOT', slot: slot({ slotId: 'other' }) });
    expect(out).toBe(s);
  });

  it('participant seeding never clobbers an edited contact name', () => {
    let s = kioskReducer(createInitialState(), {
      type: 'SET_CONTACT',
      contactInfo: { fullName: 'Typed Name', email: '', phone: '' },
    });
    s = kioskReducer(s, {
      type: 'SET_PARTICIPANTS', stations: ['A'], participantNames: [['First Participant']],
    });
    expect(s.contactInfo.fullName).toBe('Typed Name');
  });

  it('participant seeding fills an untouched contact name', () => {
    const s = kioskReducer(createInitialState(), {
      type: 'SET_PARTICIPANTS', stations: ['A'], participantNames: [['First Participant']],
    });
    expect(s.contactInfo.fullName).toBe('First Participant');
  });

  it('GO_BACK walks the side flow instead of no-opping', () => {
    const fromLookup = kioskReducer(atStep('booking-lookup'), { type: 'GO_BACK' });
    expect(fromLookup.step).toBe('service-select');
    const fromEdit = kioskReducer(
      atStep('booking-edit', { lookupBooking: {} as never }), { type: 'GO_BACK' });
    expect(fromEdit.step).toBe('booking-lookup');
    expect(fromEdit.lookupBooking).toBeNull();
  });

  it('GO_BACK is a no-op on the entry step', () => {
    const s = atStep('service-select');
    expect(kioskReducer(s, { type: 'GO_BACK' })).toBe(s);
  });

  it('UPDATE_QUANTITY clamps at zero', () => {
    const s = atStep('equipment', {
      cart: [{ item: { id: 'i1', name: 'x', prices: {} }, quantity: 2 }],
    });
    const out = kioskReducer(s, { type: 'UPDATE_QUANTITY', itemId: 'i1', quantity: -5 });
    expect(out.cart[0]?.quantity).toBe(0);
  });

  it('RESET regenerates the sessionId', () => {
    const s = createInitialState();
    const out = kioskReducer(s, { type: 'RESET' });
    expect(out.sessionId).not.toBe(s.sessionId);
  });

  it('is pure: double invocation, no input mutation', () => {
    const s = Object.freeze(atStep('summary', {
      selectedSlot: slot(),
      selectedStations: Object.freeze(['A', 'B']) as unknown as string[],
      participantNames: [['Ana'], ['Ben']],
    }));
    const action = { type: 'REFRESH_SLOT', slot: slot({ takenStations: ['A'] }) } as const;
    const a = kioskReducer(s, action);
    const b = kioskReducer(s, action);
    expect(a).toEqual(b);
    expect(s.selectedStations).toEqual(['A', 'B']);
  });
});
```
