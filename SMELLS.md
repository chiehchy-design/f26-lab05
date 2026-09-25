# reservation-service: Smells and One Fix

---

## Milestone 1: Three smells

### Smell 1: Duplicated Code

**The smell.** Duplicated Code. The pricing rule is written twice, in two files.

**Classic or agent-specific.** Agent-specific. The agent wrote a second copy instead of
reusing the first one, probably because it didn't see the existing one. It even renamed the
constants (`PREMIUM_MULTIPLIER` vs `PREMIUM_RATE_MULTIPLIER`).

**Where in the code.** `reservationManager.ts` → `calculatePrice`, and
`reportGenerator.ts` → `priceOf`.

**The principle it violates.** Duplicated Code

**What it makes expensive.** Changing a price. If someone updates only one copy, bookings
and the revenue report show different prices, and no test catches it.

### Smell 2: Speculative Generality / Dead Code

**The smell.** A cache that is set up but never used. `listBookingsForRoom` reads from the
cache, but nothing ever writes to it or clears it.

**Classic or agent-specific.** Agent-specific. The agent built half a feature. It compiles
and tests pass, so nobody noticed.

**Where in the code.** `reservationManager.ts` → `listBookingsForRoom`, plus the `cache/`
folder.

**The principle it violates.** Dead Code

**What it makes expensive.** If someone "finishes" the cache later, cancelled bookings would
still show as confirmed for 30 seconds, because nothing clears the cache.

### Smell 3: God Class

**The smell.** God Class. `ReservationManager` does too many different jobs: bookings,
pricing, sending emails, and formatting text.

**Classic or agent-specific.** Classic.

**Where in the code.** `reservationManager.ts` → `formatReceipt`, `formatDailySummary`,
`formatClock`, `formatMoney`, `dispatchNotification`.

**The principle it violates.** Single Responsibility Principle.

**What it makes expensive.** Changing how a receipt looks (for example, the currency) means
editing the same class that handles booking rules. Also, the receipt text is the email body,
so changing one changes the other.

---

## Milestone 2: One small fix

**Which smell you attacked.** Smell 1 (Duplicated Code). It can cause wrong numbers, and
the fix is small.

**What changed.**
- New file `src/pricing.ts` with one `calculatePrice` function.
- `reservationManager.ts` and `reportGenerator.ts` now both call it.
- Removed the two old copies. The math is exactly the same.

**What you deliberately did not touch.** Scope line: *only remove the duplicate pricing,
nothing else.*
- I kept `ReservationManager.calculatePrice` so nobody calling it breaks.
- I did not make the report use the saved booking price. That would change behavior.
- I did not merge the three overlap checks. That's a separate fix.

**How you know behavior is preserved.** All 39 tests pass, typecheck passes, and no test was
changed. The tests check each price rule and the revenue totals. They would not catch a
booking that is premium, long, and evening all at once, since no test has that case.

---

## Milestone 3: Two proposals and one false positive

### Proposal A (not coded): fix the cache (Smell 2)

**The problem.** The cache is half-built and will cause stale data if someone finishes it.

**The decomposition.** Move the cache out of `ReservationManager` into a new
`CachingStorageProvider` that wraps the normal storage. It clears the cache whenever a
booking is saved or updated, so it can't be forgotten. (Or just delete the cache if we don't
need it.)

**One cost.** It adds an extra layer on every storage call.

### Proposal B (not coded): split the God Class (Smell 3)

**The problem.** `ReservationManager` does too many jobs.

**The decomposition.**
- `ReservationManager`: only bookings and rooms.
- `ReceiptFormatter`: all the text formatting.
- `BookingNotifier`: sends the emails.

**One cost.** Code that calls `manager.formatReceipt()` today has to change.

### The thing that looks smelly but is fine: Long Method

**What it is.** `validation.ts` → `validateReservationRequest`. It's long, with a lot of
`if`s in a row.

**Why it is fine.** Each `if` checks one rule and returns one message. They don't depend on
each other, and there's no nesting. It doesn't change any data. It's easy to read top to
bottom, and adding a rule is just one more `if`.

**What would flip your verdict.** If the rules started changing per building (like
different opening hours), or if we needed to show all errors at once instead of the first
one. Then it should become a list of rules.
