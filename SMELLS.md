# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

The cause labels below are inferred from the code, not verified generation history.

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

**Which smell you attacked.** And why that one.

**What changed.** Files and methods you touched, and what the code does differently now.

**What you deliberately did not touch.** Name the scope line you drew and why you drew it
there. "I ran out of time" is not a scope line.

**How you know behavior is preserved.** Point at the suite, say what it actually covers, and
say what it would not catch.

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
