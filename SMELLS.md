# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1: Duplicated pricing rules (strongest)

**The smell.** Duplicated code / divergent copies of one business rule. The pricing policy
(base rate, 1.15 premium surcharge, 0.9 discount at 180+ minutes, 0.95 discount from 17:00)
is written out twice, with the constants renamed in the second copy
(`PREMIUM_MULTIPLIER` vs `PREMIUM_RATE_MULTIPLIER`, `LONG_BOOKING_MINUTES` vs
`LONG_BOOKING_CUTOFF`, `EVENING_START_MINUTE` vs `EVENING_CUTOFF`).

**Classic or agent-specific.** Agent-specific: re-implementing instead of reusing. The agent
writing `ReportGenerator` needed a price, did not find (or did not look for) the one in
`ReservationManager`, and regenerated it from the spec. The renamed constants are the
fingerprint: a human copy-pasting would have kept the names.

**Where in the code.** `src/reservationManager.ts` `calculatePrice` + `applyDiscounts`, and
`src/reportGenerator.ts` `priceOf` (plus the five constants at the top of each file).

**The principle it violates.** DRY / single source of truth: one business rule should live
in exactly one place.

**What it makes expensive.** Any pricing change, for example making the evening discount 0.9,
needs two edits in two files. If someone makes only one, receipts and revenue reports silently
disagree, and no test catches it: `reporting.test.ts` only checks that revenue matches today's
prices. It is also already semantically fragile: `revenue()` re-prices bookings from the
*current* room rate instead of reading the stored `booking.priceCents`, so re-registering a
room with a new rate rewrites past revenue.

### Smell 2: A cache that never caches

**The smell.** Speculative generality / dead infrastructure. `ReservationManager` builds a
`QueryCache` and reads from it in `listBookingsForRoom`, but nothing ever calls `cache.set`, so
every lookup misses and falls through to storage. `cacheConfig.ts` also exports `withTtl` and
`disabled`, which nothing uses.

**Classic or agent-specific.** Agent-specific: plausible scaffolding that looks finished. It
has a config object, TTL, eviction and a read path, so it reads like a working feature, but
it is only half wired.

**Where in the code.** `src/reservationManager.ts` constructor and `listBookingsForRoom`;
`src/cache/queryCache.ts`, `src/cache/cacheConfig.ts`.

**The principle it violates.** YAGNI (no speculative generality), and the code does not
honestly show its own behavior: a reader has to trace the whole class to learn the cache is
inert.

**What it makes expensive.** The first person who "finishes" it by adding `cache.set` ships a
stale-read bug. Neither `createBooking` nor `cancelBooking` invalidates `bookings:<roomId>`, so
for up to 30 s `listBookingsForRoom` and `formatDailySummary` would show a cancelled booking as
confirmed, or hide a new one. Until then it is still extra code every reader has to understand
and every change has to keep compiling.

### Smell 3: No type for a time range (primitive obsession)

**The smell.** Primitive obsession. A booking's time window is two loose `number`s
(`start`, `end`) everywhere, so every module re-derives interval logic by hand. The
"do two intervals overlap" test is written three different ways, and "clip to a window" twice.

**Classic or agent-specific.** Classic.

**Where in the code.** `src/reservationManager.ts` `hasConflict` (two early returns);
`src/availability.ts` `isSlotFree` (negated form) and `freeMinutes` (clip with
`Math.max`/`Math.min`); `src/reportGenerator.ts` `overlapsWindow` (max < min form) and the
clipping loop in `occupancy` (ternaries). `types.ts` has no range type.

**The principle it violates.** Missing domain abstraction / information hiding. The rule
"half-open intervals, touching endpoints do not overlap" has no single owner.

**What it makes expensive.** Changing the interval semantics, for example adding a 15 minute
cleaning buffer between bookings, or supporting multi-day bookings, means finding and editing
every hand-rolled copy consistently. Miss one and availability says a slot is free while
`createBooking` rejects it. Argument-order bugs (`start, end, start, end` as four bare numbers)
are also invisible to the typechecker.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Smell 1, the duplicated pricing rules. It is the one with a
real correctness risk (receipts and revenue can drift apart), and the fix is mechanical: the
two copies compute the same thing in the same order, so they can collapse into one without
any judgment call about behavior.

**What changed.**
- New `src/pricing.ts` with one exported `calculatePrice(room, start, end)` and the five
  pricing constants. It is the only place the policy lives now.
- `src/reservationManager.ts`: the five constants and the private `applyDiscounts` are gone.
  The public `calculatePrice` method stays with the same signature and delegates to `pricing.ts`.
- `src/reportGenerator.ts`: the five renamed constants and the private `durationOf` (only used
  by pricing) are gone. `priceOf` delegates to `pricing.ts`.
- The rounding order is unchanged: round the base, then after premium, after long booking,
  after evening. Both originals already rounded at every step in that order, so every input
  gives the same number as before.

**What you deliberately did not touch.** The scope line is *one owner for the pricing rule,
same outputs for every input.*
- I did not switch `ReportGenerator.revenue()` to read the stored `booking.priceCents`. That
  is arguably more correct (revenue should report what was charged, not re-price at today's
  rate), but it changes what the report means whenever a room's rate changes. That is a
  behavior change needing its own decision and its own test, not a de-duplication.
- I kept `ReservationManager.calculatePrice` as a public method instead of deleting it,
  because it is part of the class's public API and callers may use it.
- I did not unify the overlap and clip helpers or touch the cache. Those are smells 2 and 3,
  and folding them in would make this diff about three things instead of one.

**How you know behavior is preserved.** `npm test` passes 39/39 with no test edited, and
`npm run typecheck` is clean. `booking.test.ts` pins the price for a plain booking, a 3 hour
or longer booking, a premium room and an evening booking, which covers each branch of
`calculatePrice`. `reporting.test.ts` checks that revenue equals the sum of the booking prices,
so both callers are exercised. What the suite would *not* catch: combined discounts (for
example premium + long + evening in one booking, where rounding order matters), or a room
whose rate changed after booking. Behavior is preserved there too, but by construction (the
same operations in the same order), not because a test checks it.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

**The problem.** Name it.

**The decomposition.** What are the pieces, what does each own, and where do the rules live?

**One cost.** Something this actually costs. "No real downside" is not a cost.

### Proposal B (not coded)

**The problem.**

**The decomposition.**

**One cost.**

### The thing that looks smelly but is fine

**What it is.** File and method.

**Why it is fine.** Defend it with properties of the code, not with its line count.

**What would flip your verdict.** Name the change that would turn this into a real problem.
