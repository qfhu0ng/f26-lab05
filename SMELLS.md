# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

These findings describe the code before the Milestone 2 fix. The cause labels below
are inferred from the code, not verified generation history.

### Smell 1: Duplication over reuse

**The smell.** Duplication over reuse: booking creation and revenue reporting each
implement the same pricing policy.

**Classic or agent-specific.** Agent-specific. The likely cause is missing context:
the reporting code reimplements pricing rather than reusing the existing rules.

**Where in the code.** `src/reservationManager.ts`: `calculatePrice()` and
`applyDiscounts()`; `src/reportGenerator.ts`: `priceOf()`. Both files also define
their own surcharge, discount, and cutoff constants.

**The principle it violates.** DRY: one business policy has two independently
maintained implementations, including the order of rounding.

**What it makes expensive.** Changing the evening discount requires edits in both
files. Updating only booking creation would make newly stored prices disagree with
the prices recomputed by the revenue report.

### Smell 2: Phantom complexity

**The smell.** Phantom complexity: the booking query has a cache lookup, but its
cache never receives any entries through the service's normal call paths.

**Classic or agent-specific.** Agent-specific. Missing context is a plausible cause:
the cache implementation was not connected to the complete read/write flow. Free
volume can explain the surrounding TTL, capacity, and invalidation machinery.

**Where in the code.** `src/reservationManager.ts`: the constructor creates a private
`QueryCache`, and `listBookingsForRoom()` calls `get()` but never `set()`.
`src/cache/queryCache.ts` and `src/cache/cacheConfig.ts` provide the unused machinery.

**The principle it violates.** Simplicity: additional state and control flow should
serve an actual requirement. This integration adds complexity without avoiding any
storage queries.

**What it makes expensive.** A maintainer investigating query performance must trace
the cache lifecycle to discover that every lookup misses. Enabling writes later also
requires deciding how creation and cancellation invalidate cached booking lists.

### Smell 3: Speculative over-abstraction

**The smell.** Speculative over-abstraction: a dynamic notification registry wraps
the service's single, fixed email channel.

**Classic or agent-specific.** Agent-specific. The likely cause is an underspecified
request: the implementation anticipates pluggable channels without a demonstrated
need for dynamic registration in the current service.

**Where in the code.** `src/notifications/notifierFactory.ts`: `ChannelName` permits
only `'email'`, yet `builders`, `registerChannel()`, and `createNotificationChannel()`
implement a mutable registry. `ReservationManager` always uses the default config.

**The principle it violates.** YAGNI: pay for extension mechanisms when a supported
variation needs them. The concern is the dynamic registry, not the mere existence of
the `NotificationChannel` interface.

**What it makes expensive.** Understanding email construction requires following
module-level registration and builder lookup. Replacing a builder for a test changes
global state that must be restored to avoid affecting later manager instances.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Smell 1, duplication over reuse. The duplicated pricing
policy has a clear boundary and can be centralized through a small,
behavior-preserving change. This removes two independently maintained
implementations without a broader redesign of booking, reporting, or storage.

**What changed.** Added `src/pricing.ts` with one pure `calculatePrice(room, start,
end)` function owning the pricing constants and calculation. In
`src/reservationManager.ts`, the existing public `calculatePrice()` delegates to it;
the private `applyDiscounts()` and duplicate constants were removed. In
`src/reportGenerator.ts`, `revenue()` calls the same function; the private `priceOf()`,
its `durationOf()` helper, and duplicate constants were removed. Both callers now
use one policy implementation.

**What you deliberately did not touch.** The scope is pricing-policy extraction.
The thresholds, adjustment order, and rounding after each step stay unchanged, as
do existing public method signatures. Revenue still recomputes prices from room
rates rather than summing stored `priceCents`; changing that would change reporting
semantics. Validation, overlap rules, storage, notification, and caching remain
unchanged. The other two smells are reserved for Milestone 3 proposals. No tests
were edited.

**How you know behavior is preserved.** Before and after the refactor, `npm test`
passed all 39 tests across three files, and `npm run typecheck` passed. The suite
checks plain pricing, the premium surcharge, long-booking and evening discounts,
revenue totals and cancellation filtering, plus booking, validation, availability,
and occupancy behavior. It does not exhaustively cover combinations of discounts
or rounding-sensitive rates. Inspection of the extraction confirms the same
arithmetic order and `Math.round` steps; the green suite alone is not a proof for
every input.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A: Remove the inactive cache layer (not coded)

**The problem.** Smell 2, phantom complexity. `ReservationManager` constructs a
private cache and reads it in `listBookingsForRoom()`, but no service path populates
it. The cache suggests an optimization that never avoids a storage read.

**The decomposition.** Keep `ReservationManager` responsible for booking operations
and make `listBookingsForRoom()` delegate directly to `StorageProvider.findByRoom()`.
Booking records remain in `InMemoryStorageProvider`'s `Map<string, Booking>`:
creation still calls `save()`, cancellation still calls `update()`, and queries read
that same storage. The provider retains responsibility for insertion order and
copying records. Remove only the redundant cache layer: the manager's cache field,
initialization, and lookup branch, plus the cache utilities after checking their
exported APIs for consumers. Booking rules stay in the manager and validator;
there is no cache invalidation policy to coordinate with them. Add caching only if
a measured query cost justifies designing its read/write lifecycle.

**One cost.** Removing the exported cache utilities requires a consumer audit and
migration for any callers outside this repository. It also gives up that reusable
TTL/eviction implementation if a later workload actually needs caching.

### Proposal B: Inject the notification channel (not coded)

**The problem.** Smell 3, speculative over-abstraction. `notifierFactory.ts` uses a
mutable global builder registry even though the service always selects email.

**The decomposition.** Keep `NotificationChannel` as the sending boundary and
`EmailChannel` responsible for email formatting and its sent-message record. Give
`ReservationManager` an optional channel constructor argument, defaulting to an
`EmailChannel`; callers can pass a configured channel or a test double explicitly.
The caller selects and configures the channel when constructing a manager. The
manager still decides when to notify and builds the receipt; the supplied channel
handles sending. Remove the global builder registry and registration-based factory.
Keep the interface because it supports substitution without shared registration
state; do not introduce a new plugin-selection mechanism.

**One cost.** Callers using `registerChannel()` would need to migrate to per-instance
wiring. Changing a channel globally through registration would no longer work;
callers needing that behavior would have to coordinate their manager construction.

### The thing that looks smelly but is fine

**What it is.** `src/validation.ts`, `validateReservationRequest()`, can look like a
long-method smell because it contains many conditional checks.

**Why it is fine.** It performs one cohesive operation: validate one request against
one room and return the first failure. It has no storage, notification, or mutation
side effects. The checks form an explicit sequence: validate the input shape and
time interval before applying duration, capacity, and building rules. Keeping that
sequence together makes error precedence visible without a rule-dispatch framework.

**What would flip your verdict.** Supporting several buildings with different
opening hours, time boundaries, and premium-room policies would make interleaved
building-specific branches harder to change independently. At that point, separate
common request checks from a selected building policy, while preserving explicit
error precedence.
