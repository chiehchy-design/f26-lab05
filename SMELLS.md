# reservation-service: Smells and One Fix

---

## Milestone 1: Three smells

### Smell 1: Duplicated Code (leads to Shotgun Surgery)

**The smell.** Duplicated Code, which causes Shotgun Surgery. The pricing rule (hourly rate, premium surcharge, long-booking
discount, evening discount) was written twice, with its own copy of the constants under
different names (`PREMIUM_MULTIPLIER` vs `PREMIUM_RATE_MULTIPLIER`, `EVENING_START_MINUTE` vs
`EVENING_CUTOFF`, ...).

**Classic or agent-specific.** Agent-specific. It looks like an agent writing `ReportGenerator`
needed a price, did not look at (or did not have in context) `ReservationManager.calculatePrice`,
and wrote a fresh copy instead of reusing it. The renamed constants are the giveaway: a human
copy-paste would keep the names.

**Where in the code.** `src/reservationManager.ts` → `calculatePrice` + `applyDiscounts`, and
`src/reportGenerator.ts` → `priceOf` + `durationOf`.

**The principle it violates.** DRY / single source of truth. One business rule should live in
one place.

**What it makes expensive.** Any price change. If the evening discount goes from 5% to 10% and
someone only edits `reservationManager.ts`, new bookings get the new price but the revenue
report still uses the old one. The two numbers silently disagree, and no test fails, because
nothing forces the two copies to match.

### Smell 2: Speculative Generality / Dead Code

**The smell.** Speculative Generality / Dead Code (a half-wired feature).
`listBookingsForRoom` reads from `QueryCache`, but nothing ever calls `cache.set`, and
`createBooking` / `cancelBooking` never call `cache.invalidate`. The cache always misses.

**Classic or agent-specific.** Agent-specific. It is plausible-looking scaffolding: the agent
added a `cache/` folder, a config object, and a read path, but never finished the write path
or the invalidation. It compiles and the tests pass, so nothing flagged it.

**Where in the code.** `src/reservationManager.ts` → `listBookingsForRoom` (and the constructor
that builds the cache); `src/cache/queryCache.ts`, `src/cache/cacheConfig.ts`.

**The principle it violates.** YAGNI (don't build what you don't use). It also breaks the rule
that whoever changes data must invalidate the cache of that data.

**What it makes expensive.** Today it is just noise a reader has to figure out. The real cost
comes the day someone "finishes" it by adding `cache.set` in `listBookingsForRoom`: after a
`cancelBooking`, `formatDailySummary` would keep showing the booking as confirmed for 30
seconds (the TTL), because nothing invalidates the key.

### Smell 3: God Class (Divergent Change)

**The smell.** God Class, which causes Divergent Change. The class's own doc comment lists five jobs:
room registry, booking lifecycle, pricing, sending notifications, and formatting receipts and
summaries.

**Classic or agent-specific.** Classic. This is the textbook "one class that changes for many
different reasons."

**Where in the code.** `src/reservationManager.ts` → `formatReceipt`, `formatDailySummary`,
`formatClock`, `formatMoney`, `dispatchNotification`, and the constructor, which hard-codes
`createNotificationChannel(DEFAULT_NOTIFIER_CONFIG)`.

**The principle it violates.** Single Responsibility Principle (and dependency inversion for
the notifier: the manager builds its own channel instead of being given one).

**What it makes expensive.** Changing how things look. Switching `$` to another currency, or
12-hour clock times, means editing the class that also enforces booking conflicts. And since
the email body *is* `formatReceipt`, a text tweak meant for a printed receipt also changes
every email. Tests also cannot pass in a fake notifier, so they can only check the
`notificationLog` strings, not what was actually sent.

---

## Milestone 2: One small fix

**Which smell you attacked.** Smell 1, Duplicated Code (pricing). It is the only one of the three
that can give wrong numbers today with a one-line edit, and it has a small, clear fix.

**What changed.**
- New file `src/pricing.ts` with one function `calculatePrice(room, start, end)` and the five
  pricing constants. The logic is the same as before, same order of steps, same rounding.
- `src/reservationManager.ts`: removed the constants and `applyDiscounts`;
  `calculatePrice` now just calls the shared function.
- `src/reportGenerator.ts`: removed the constants, `priceOf`, and `durationOf`; `revenue`
  now calls the shared function.

Now there is one copy of the pricing rule.

**What you deliberately did not touch.** My scope line: *remove the duplicate pricing rule
and nothing else.*
- I kept `ReservationManager.calculatePrice` as a public method (it just delegates), so no
  caller's API changes.
- I did **not** change `ReportGenerator.revenue` to use the stored `booking.priceCents`
  instead of recomputing. That might be more correct, but it changes behavior (old bookings
  would report the price they were sold at, not today's price). That is a product decision,
  not a refactor.
- I did **not** merge the overlap checks (`hasConflict`, `isSlotFree`, `overlapsWindow`).
  That is the same kind of smell, but a different rule in different files (availability,
  booking, reports), so it would be a second fix, not part of this one.
- I did not touch smells 2 and 3.

**How you know behavior is preserved.** `npm test` is 39/39 green before and after, and
`npm run typecheck` passes. No test was edited. The suite checks each pricing rule on its own
through `createBooking` (plain, long, premium, evening), and the revenue tests check that the
report total equals the sum of booking prices, so both callers are exercised.
What it would *not* catch: no test combines premium + long + evening in one booking, so a
change to the order of rounding steps could slip through. I kept the order exactly the same
to avoid that.

---

## Milestone 3: Two proposals and one false positive

### Proposal A (not coded): Speculative Generality in the cache (Smell 2)

**The problem.** A cache that is read but never written or invalidated. It is dead today, and
it becomes a stale-data bug the moment someone finishes it in the obvious place.

**The decomposition.** Take caching out of `ReservationManager` entirely.
- `ReservationManager` only talks to a `StorageProvider`. No cache field.
- New `CachingStorageProvider implements StorageProvider`, which wraps another provider plus
  a `QueryCache`. It owns the rule: `findByRoom` reads through the cache; `save`, `update`,
  and `clear` invalidate the affected room key.
- Whoever builds the service chooses `new CachingStorageProvider(new InMemoryStorageProvider())`
  or the plain one.

The invalidation rule then lives next to the writes, so it cannot be forgotten.
(If nobody actually needs caching, the cheaper option is just deleting it.)

**One cost.** An extra layer that every storage call goes through, and cached arrays are
shared. `InMemoryStorageProvider` returns fresh copies each call, but a cache would return
the same array, so a caller that mutates the result would corrupt the cache unless the
wrapper copies it too. That means more copying, or a new rule callers must follow.

### Proposal B (not coded): break up the God Class ReservationManager (Smell 3)

**The problem.** One class owns booking rules, presentation text, and notifier setup, so
unrelated changes all land in the same file.

**The decomposition.**
- `ReservationManager`: rooms and the booking lifecycle (create, cancel, conflicts, slots).
  It gets a `NotificationChannel` passed into its constructor (default still email).
- `ReceiptFormatter` (new): `formatReceipt`, `formatDailySummary`, `formatClock`,
  `formatMoney`. Owns all the text and number formatting.
- `BookingNotifier` (new, small): takes a channel and a formatter, sends
  "Reservation confirmed/cancelled", keeps the log.

Booking rules live in the manager, text rules live in the formatter, and delivery rules live
in the channel.

**One cost.** The conflict error message in `createBooking` uses `formatClock`, so the manager
still needs the clock helper, either by depending on the formatter or by moving `formatClock`
into a shared util. Also `formatReceipt` / `formatDailySummary` are public on the manager
today, so every caller must change, or we keep delegating methods on the manager (which
leaves part of the smell in place).

### The thing that looks smelly but is fine: looks like a Long Method

**What it is.** `src/validation.ts` → `validateReservationRequest`. About 50 lines, around a
dozen `if` statements in a row, which at first glance looks like a long method.

**Why it is fine.**
- It is a pure function: it reads the request and room, touches no state, calls nothing else.
- Every rule is one flat guard clause with its own message. There is no nesting, and no rule
  depends on another rule's result.
- It returns on the first failure on purpose (the doc comment says so), so the order is the
  priority. Reading top to bottom *is* the spec.
- The tests (`tests/validation.test.ts`) are one table row per rule, which lines up 1-to-1
  with the `if`s.

Splitting it into a rule-object list or a class per rule would add indirection without making
anything easier to change. Adding a rule today is just one more `if`.

**What would flip your verdict.** If the rules started to *vary*. For example, each building
having its own opening hours or boundary size, premium rules differing per room type, or the
UI wanting *all* errors at once instead of the first. Then the hard-coded constants and the
early returns become the problem, and a list of rule objects (per building, all run) would be
worth it. The same goes if a rule needed storage (e.g. "max 3 bookings per organizer per
day"), because the function would stop being pure.
