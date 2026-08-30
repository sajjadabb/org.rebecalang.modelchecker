# Testing notes — transparentactormodelchecker

Working document for the unit-test work on the Transparent Actor Model Checker.
It records what is covered, what the suite currently reports, which defects the
tests exposed, and what is planned next. Kept up to date as the work proceeds.

**Status:** tests landed; four defects fixed. The whole suite passes its assertions;
what remains are four errors, all of them stack overflows or a null hash on models
large enough to reach code that was previously never executed.
**Last updated:** 2026-08-30 · branch `tests/transparent-actor-coverage`

---

## 1. Scope

Work is confined to `org.rebecalang.transparentactormodelchecker`, the newer of the
two engines in this repository. The older `modelchecker` engine is untouched, and so
is `CoreRebecaModelsTest`, which is byte-identical to `master`.

Four small changes have been made to `src/main`, all listed in section 7. Everything
else is a test.

## 2. Building and running

The `org.rebecalang` artifacts are not published to Maven Central, and the poms pin
the sibling dependencies as `LATEST`, so the two dependencies must be installed into
the local repository first, in this order:

```
mvn -f org.rebecalang.compiler/pom.xml         install -DskipTests   # 2.31
mvn -f org.rebecalang.modeltransformer/pom.xml install -DskipTests   # 1.13
mvn -f org.rebecalang.modelchecker/pom.xml     test                  # 4.0
```

Requires JDK 17. Verified against `compiler` `dc663e0`, `modeltransformer` `2a1d0bd`
and `modelchecker` `fb4197f4`.

## 3. Baseline, before any test was added

Recorded on `master` so that later claims can be measured against something:

```
Tests run: 44, Failures: 3, Errors: 3, Skipped: 5
```

| Failure | Cause |
|---|---|
| `CoreRebecaModelsTest.GIVEN_RebecaModel_WHEN_No_Error` | `NullPointerException` in `ActivationRecord.hashCode` — it dereferences entry values unguarded while its own `equals` handles `null` |
| `CoreRebecaModelsTest.GIVEN_DiningPhilosopherModel...[3]`, `[4]` | `StackOverflowError` from unbounded recursion `dfs` → `deliverAllMessagesAndStore` → `dfs` |
| `TimedRebecaModelsTest.ticketService[1..3]` | state space collapses: expected 5 / 345 / 13723, got 3 / 5 / 7 |

The 5 skipped tests are the `@Disabled` legacy-engine tests.

## 4. What the tests cover

53 tests were added across five classes.

| Test class | Before | After |
|---|---:|---:|
| `corerebeca/ActorScopeTest` | 2 | 19 |
| `corerebeca/StatementsSOSRulesTest` | 7 | 17 |
| `corerebeca/NetworkAndActorSOSRulesTest` | 2 | 10 |
| `timedrebeca/TimedNetworkTest` | 5 | 13 |
| `timedrebeca/TimedRebecaMessageTest` | 4 | 14 |
| **suite total** | **44** | **97** |

### 4.1 `ActorScopeTest` — variable scoping

`ActorScope` has fourteen public methods and had two tests. The added tests cover:

- **Name lookup** — a bound name is found, an unbound one is not, and reading an
  unbound name raises `RebecaRuntimeInterpreterException`.
- **Scope stack** — a variable declared in an inner frame disappears when the frame
  is popped while the enclosing one survives; an inner frame shadows an outer name
  and the outer value returns after the pop.
- **Method-call frames** — a call frame jumps back to the actor's own frame, so the
  caller's block locals are not visible to the callee; `popToReturn` writes the
  return value into the named variable and discards any frames opened inside the call.
- **An actor as a variable of another actor** — an actor stored in scope is kept by
  id (`ActorStateRepresentor`) and resolved back through the environment's actor
  container, so reading it returns the same actor state. Nested access through the
  peer works for both a plain field (`peer.x`) and an array element (`peer.arr[1]`),
  for reading and for assignment, and an assignment lands in the peer's own scope.
- **Cloning and identity** — a clone is independent of the original; two scopes with
  equal frames are equal even when their environments differ, because the environment
  is shared between actors and must not take part in state identity.

One behaviour worth knowing: name lookup walks the scope stack down to index 0, which
holds the shared environment and is `null` in a freshly constructed `ActorScope`.
Looking up an absent name before an environment is attached therefore raises
`NullPointerException` rather than the intended interpreter exception.

### 4.2 `StatementsSOSRulesTest` — statement-level SOS rules

Six of the eleven statement-level rules had no test at all. All six are now covered:

| Rule | What the tests establish |
|---|---|
| `PushRule` | opens a new frame and advances the program counter |
| `PopRule` | pops exactly `numberOfPops` frames — verified for one and for two |
| `MethodCallRule` | enters the callee with its own program counter at the resolved body; a literal argument is bound as-is and a variable argument is evaluated in the caller |
| `EndMethodCallRule` | restores the caller's program counter, on the instruction after the call |
| `EndMSGSrvRule` | drops the message-server frame while the actor's own fields survive |
| `SendMessageRule` | addresses the named peer, falls back to self when no receiver is named, and carries an actor-valued argument by id rather than by reference |

`SendMessageRule` is the only statement-level rule without `@Component`, so it is not
a Spring bean and the test constructs it directly. `MethodCallRule` keeps its lookup
table in a plain field with a setter rather than an injected one, so the test sets it
explicitly; calling the rule without doing so fails with a bare `NullPointerException`.

### 4.3 `NetworkAndActorSOSRulesTest` — actor- and network-level rules

The existing two tests exercise the composition level. The added tests go one level
down, to the individual rules.

- `ReceiveMessageRule` places the message in the actor's queue and reports the same
  action it delivered.
- `CoreRebecaTakeMessageRule` prepares the message-server frame: the sender is bound,
  the message parameters enter scope, and the program counter is set to the start of
  the message server. It raises `RuleIsDisabledException` on an empty queue, and its
  enablement check refuses an actor that is already mid-execution.
- `NetworkReceiveMessageRule` groups messages by the sender/receiver route.
- `CoreRebecaNetworkLevelDeliverMessageRule` removes the delivered message from the
  network, is disabled on an empty network, and — the point worth recording —
  **produces one branch per route**, so delivery is non-deterministic and the checker
  explores each ordering.

`TakeMessageRule` and `NetworkLevelDeliverMessageRule` are abstract (the latter has no
members at all), so the tests target the concrete Core Rebeca subclasses.

### 4.4 Timed extension

`TimedRebecaMessageState` and the actor's message bag:

- shift equivalence for an identical message (shift 0), for a later one, and for an
  earlier one (negative shift);
- non-equivalence when arrival and deadline move by different amounts, so the
  remaining slack differs, and when the receiver differs;
- cloning copies arrival and deadline and is independent of the original;
- the bag is ordered by arrival on receive, `bagIsEmpty` and `messageQueueIsEmpty`
  track its contents, and `getFirstMessageArrivalTime` reports the head;
- **only the first message of any given sender is enabled**, so a sender's stream
  keeps its order;
- `getEnableMessage` removes the message rather than peeking at it.

`TimedRebecaNetworkState`: messages sharing an arrival time share one time bucket,
distinct arrivals create separate buckets, two empty networks are shift-equivalent,
and networks with different bucket counts are not.

## 5. Current results

```
Tests run: 97, Failures: 0, Errors: 4, Skipped: 5
```

Every assertion in the suite passes. The four errors are all crashes on the larger
models, described in sections 6 and 7.

How the suite moved:

| Stage | Tests | Failures | Errors |
|---|---:|---:|---:|
| `master`, before any work | 44 | 3 | 3 |
| after the tests were added | 97 | 6 | 3 |
| after the first three fixes | 97 | 3 | 3 |
| after the `delay` fix | 97 | **0** | 4 |

Every added assertion was checked by inverting it and confirming that the test then
fails. A test that stays green when its assertion is reversed is not testing anything.
Two rounds were needed in places: when two mutations land in the same test method only
the first is exercised, because the first failing assertion aborts the rest.

## 6. Defects found

Each was confirmed with a minimal failing test or a direct measurement before being
recorded here. The reproductions are kept as regression tests.

### 6.1 `TimeBucket.clone()` discarded every message

```java
TimeBucket timeBucket = new TimeBucket(time);
for (Entry<Integer, ActorReceivingBucket> arBucket : timeBucket.messages.entrySet()) {
```

The loop iterated `timeBucket.messages` — the newly created, empty map — instead of
`this.messages`, so its body never ran. The clone kept the right `time` and no messages.
`TimedRebecaNetworkState.clone()` calls this for every bucket, so cloning a timed network
state silently emptied it.

Reproduced by `GIVEN_ANetworkHoldingAMessage_WHEN_ItIsCloned_THEN_TheCloneStillHoldsThatMessage`
— `expected: <1> but was: <0>`.

### 6.2 `ActorReceivingBucket.shiftEquals()` compared only the first message

```java
for (int cnt = 0; cnt < this.sentMessages.size(); cnt++)
    thisMessages.get(cnt).shiftEquals(otherMessages.get(cnt));
```

The loop was bounded by `sentMessages.size()`, the number of receiver keys, while the
index was applied to `thisMessages`, the message list for one key. Every
`ActorReceivingBucket` is reached through `TimeBucket.getReceiverMessages(receiverId)`
and therefore holds messages for exactly one receiver, so the bound was always 1 and
every message after the first was never compared.

Reproduced by `GIVEN_AnActorReceivesTwoMessagesAtOneInstant_WHEN_OnlyTheSecondDiffers_THEN_TheNetworksAreNotEquivalent`
— `expected: <false> but was: <true>`.

### 6.3 `TimedRebecaNetworkState.addMessage()` did not keep buckets in time order

The `time > arrivalTime` case fell through both conditions and the loop continued, so it
behaved exactly like `time < arrivalTime`. The scan never stopped at the insertion point,
`cnt` always ended at `size()`, and `add(cnt, timeBucket)` degenerated into an append —
the `add(index, …)` call shows sorted insertion was intended. Bucket order then followed
insertion order, while `equals` and `shiftEquals` compare the two lists index by index.

Reproduced by `GIVEN_AMessageArrivesEarlierThanAnExistingOne_WHEN_ItIsAdded_THEN_BucketsStayOrderedByTime`
— `expected: <10> but was: <30>`.

### 6.4 The built-in `delay` method was registered under an unreachable name

This is the one that made `ticketService` report a collapsed state space.

`TransparentActorTimedRebecaFTTSModelChecker.initializeMethodBindingTable()` registers the
built-in `delay` through `RILUtilities.computeMethodName(null, "delay", …)`, and that
method builds its result as `className + "." + methodName`. With a `null` class name Java
renders the literal string `"null"`, so the entry went in as `null.delay$int`. The RIL
emits a base-less call as plain `delay$int`. Measured directly:

```
REGISTERED = [null.delay$int]
CALLED     = [delay$int]
```

`MethodLookup.resolveName` therefore returned `null`, `MethodCallRule` stored a program
counter of `(null, 0)`, and the next `getEnabledInstruction()` called
`RILModel.getInstructionList(null)`, which is a `Hashtable.get(null)` —
`NullPointerException`.

Every Timed Rebeca model that calls `delay` crashed the moment it reached that statement.
In the ticket service the crash happens in the third state, which is why the reported
state space was 3 instead of 5.

### 6.5 A crash inside model checking is reported as a result

```java
try {
    dfs(initialState);
} catch (Exception e) {
    e.printStackTrace();
    result = new TransparentActorModelCheckingResult(INTERNAL_ERROR);
    result.setTransitionSystem(transitionSystem);   // the partial one
    return result;
}
```

`modelcheck` catches everything, prints a stack trace, and returns the partially built
transition system. Callers that only read `getTransitionSystem().size()` — including
`TimedRebecaModelsTest` — get a plausible-looking number that is simply the point where
the search died.

This is worth separating from 6.4. The name-resolution bug was a crash; this is what
turned the crash into a wrong answer. Because of it, the symptom read as "the state
space collapses" and pointed the investigation at state merging, which cost a full round
of work before the real cause was measured. It is not fixed here, since deciding what a
model checker should return when it fails is the supervisor's call, but a caller cannot
currently distinguish a completed search from an aborted one without checking the result
status.

## 7. Changes applied to `src/main`

Four changes, each tied to a defect above.

| # | Change | Effect |
|---|---|---|
| 1 | `TimeBucket.clone()` — iterate `this.messages` | 6.1 reproduction green |
| 2 | `ActorReceivingBucket.shiftEquals()` — bound the loop by `thisMessages.size()` | 6.2 reproduction green |
| 3 | `TimedRebecaNetworkState.addMessage()` — stop at the insertion point, keep buckets in time order | 6.3 reproduction green |
| 4 | `TransparentActorTimedRebecaFTTSModelChecker` — register `delay` under the name the RIL actually calls | `ticketService` reaches its expected state counts |

Fix 3 needed more than an added `break`. The check after the loop,
`if (cnt != receivedMessages.size())`, could not tell "found a bucket at the same time"
from "found the position to insert before". Breaking on `time > arrivalTime` alone would
have made it reuse a bucket belonging to a *later* time — worse than the original bug. A
flag now separates the two cases.

### What the first three fixes did to `ticketService`: nothing

6.2 was put forward as a plausible cause of the collapsing state space, on the grounds
that over-aggressive state merging produces exactly that shape. It was wrong. After
fixes 1–3 the numbers were byte-for-byte what they had been:

| case | expected | before | after fixes 1–3 |
|---|---:|---:|---:|
| [1] | 5 | 3 | 3 |
| [2] | 345 | 5 | 5 |
| [3] | 13723 | 7 | 7 |

Not a single state moved. That ruled the hypothesis out and sent the investigation back
to measurement rather than reading: dumping the three-state transition system showed the
search stopping precisely before `TicketService.requestTicket`, whose body is the only
place the model calls `delay`. Printing the registered name against the called name gave
6.4 directly.

### What fix 4 did

| case | expected | before | after |
|---|---:|---:|---:|
| [1] | 5 | 3 | **5** |
| [2] | 345 | 5 | **345** |
| [3] | 13723 | 7 | `StackOverflowError` |

The first two now reach their expected counts exactly. The third explores far enough to
exhaust the JVM stack in the recursive `dfs` / `deliverAllMessagesAndExpand` pair.

That third result is progress, not a regression. Before the fix the model was never
explored at all: the search died after three states and returned 7 as though it were an
answer. It now runs until it genuinely runs out of stack — a visible failure instead of a
quiet wrong number.

The remaining four errors are all of that kind:

| Error | Note |
|---|---|
| `CoreRebecaModelsTest.GIVEN_RebecaModel_WHEN_No_Error` | `NullPointerException` in `ActivationRecord.hashCode`, which dereferences entry values unguarded while its own `equals` handles `null` |
| `CoreRebecaModelsTest.GIVEN_DiningPhilosopherModel…[3]`, `[4]` | `StackOverflowError` in the recursive depth-first search |
| `TimedRebecaModelsTest.ticketService[3]` | same recursion, now reachable |

Three of the four were already failing on `master`. All four are untouched by the changes
above and would need either an iterative search or a larger stack.

## 8. Open questions

- Should the recursive `dfs` be rewritten with an explicit work list? Three of the four
  remaining errors are stack overflows, and the fourth model is only moderately large.
- Should `modelcheck` still return a transition system when it aborts (6.5), or should
  the failure reach the caller?
- Would `RILUtilities.computeMethodName` be better off treating a `null` class name as
  "no class" rather than rendering the string `"null"`? That is the upstream form of 6.4
  and would close the trap for every future caller, but it lives in
  `org.rebecalang.modeltransformer` and affects everything that calls it.
- In `TimedRebecaActorState.receiveMessage`, messages arriving at the same instant are
  ordered by *descending* name. Any deterministic order is sound, so this may well be
  intentional.
- `SendMessageRule` lacks `@Component` while the other ten statement-level rules have it:
  deliberate because `TimedRebecaSendMessageSOSRule` extends it, or an oversight?
- Should `ActorScope` name lookup tolerate a missing environment instead of raising
  `NullPointerException`?

## 9. Still uncovered

Known gaps, in rough order of value:

- `TimedRebecaActorState`: `memoizedClone`, `isEnable`, `createNewActorState`.
- `TimedRebecaSystemState` and `TimedActorScope` have no direct tests.
- `ActorReceivingBucket` is exercised only through the classes above it.
- The timed variants of the network delivery rules
  (`TimedRebecaNetworkLevelDeliverMessage`, `TimedRebecaFTTSNetworkLevelDeliverMessage`).

## 10. Log

| Date | Change |
|---|---|
| 2026-08-30 | Baseline recorded (44 tests, 3 failures, 3 errors). 53 tests added across five classes; suite at 97. Three defects found and reproduced. `src/main` unchanged. |
| 2026-08-30 | The three defects fixed; their reproductions pass and the suite is at 97 / 3 / 3. `ticketService` unchanged, ruling 6.2 out as its cause. |
| 2026-08-30 | `ticketService` root-caused to 6.4 by measurement and fixed. Suite at 97 / 0 / 4: all assertions pass, and the remaining errors are crashes on large models, three of which predate this work. |
