# Testing notes — transparentactormodelchecker

Working document for the unit-test work on the Transparent Actor Model Checker.
It records what is covered, what the suite currently reports, which defects the
tests exposed, and what is planned next. Kept up to date as the work proceeds.

**Status:** tests landed, production code not yet changed.
**Last updated:** 2026-08-30 · branch `tests/transparent-actor-coverage`

---

## 1. Scope

Work is confined to `org.rebecalang.transparentactormodelchecker`, the newer of the
two engines in this repository. The older `modelchecker` engine is untouched, and so
is `CoreRebecaModelsTest`, which is byte-identical to `master`.

No file under `src/main` has been modified. Every change so far is a test.

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
Tests run: 97, Failures: 6, Errors: 3, Skipped: 5
```

Three failures and three errors are the pre-existing ones from section 3. The other
three failures are new and deliberate — see the next section.

Every added assertion was checked by inverting it and confirming that the test then
fails. A test that stays green when its assertion is reversed is not testing anything,
and two rounds were needed in places: when two mutations land in the same test method
only the first is exercised, because the first failing assertion aborts the rest.

## 6. Defects the tests exposed

All three were found by reading the code and then confirmed with a minimal failing
test. They are left failing on purpose, so they stay visible rather than being
asserted away.

### 6.1 `TimeBucket.clone()` discards every message

```java
TimeBucket timeBucket = new TimeBucket(time);
for (Entry<Integer, ActorReceivingBucket> arBucket : timeBucket.messages.entrySet()) {
```

The loop iterates `timeBucket.messages` — the newly created, empty map — instead of
`this.messages`, so its body never runs. The clone keeps the right `time` and no
messages. `TimedRebecaNetworkState.clone()` calls this for every bucket, so cloning a
timed network state silently empties it.

Reproduced by `GIVEN_ANetworkHoldingAMessage_WHEN_ItIsCloned_THEN_TheCloneStillHoldsThatMessage`
— `expected: <1> but was: <0>`.

### 6.2 `ActorReceivingBucket.shiftEquals()` compares only the first message

```java
for (int cnt = 0; cnt < this.sentMessages.size(); cnt++)
    thisMessages.get(cnt).shiftEquals(otherMessages.get(cnt));
```

The loop is bounded by `sentMessages.size()`, the number of receiver keys, but the
index is applied to `thisMessages`, the message list for one key. Every
`ActorReceivingBucket` is reached through `TimeBucket.getReceiverMessages(receiverId)`
and therefore holds messages for exactly one receiver, so the bound is always 1 and
every message after the first is never compared.

Two states that differ only in a later message at the same instant are reported as
equivalent, and the search treats one as already visited.

Reproduced by `GIVEN_AnActorReceivesTwoMessagesAtOneInstant_WHEN_OnlyTheSecondDiffers_THEN_TheNetworksAreNotEquivalent`
— `expected: <false> but was: <true>`.

### 6.3 `TimedRebecaNetworkState.addMessage()` does not keep buckets in time order

```java
for (; cnt < receivedMessages.size(); cnt++) {
    int time = receivedMessages.get(cnt).getTime();
    if (time < arrivalTime) continue;
    if (time == arrivalTime) break;
}
```

The `time > arrivalTime` case falls through both conditions and the loop continues, so
it behaves exactly like `time < arrivalTime`. The scan never stops at the insertion
point, `cnt` always ends at `size()`, and `add(cnt, timeBucket)` degenerates into an
append — the `add(index, …)` call shows sorted insertion was intended. Bucket order
then follows insertion order, while `equals` and `shiftEquals` compare the two lists
index by index.

Reproduced by `GIVEN_AMessageArrivesEarlierThanAnExistingOne_WHEN_ItIsAdded_THEN_BucketsStayOrderedByTime`
— `expected: <10> but was: <30>`.

## 7. Planned changes to `src/main`

Not yet applied. To be agreed with the supervisor first, since this area is under
active development.

| # | Change | Test that should turn green |
|---|---|---|
| 1 | `TimeBucket.clone()` — iterate `this.messages` | `…_TheCloneStillHoldsThatMessage` |
| 2 | `ActorReceivingBucket.shiftEquals()` — bound the loop by `thisMessages.size()` | `…_TheNetworksAreNotEquivalent` |
| 3 | `TimedRebecaNetworkState.addMessage()` — stop the scan when `time > arrivalTime` so the bucket is inserted in order | `…_BucketsStayOrderedByTime` |

Expected suite afterwards:

```
Tests run: 97, Failures: 0-3, Errors: 3
```

The three deliberate failures should clear. Whether `TimedRebecaModelsTest.ticketService`
also recovers is **an open question, not a prediction**. Its symptom — a state space
that shrinks further the larger the model gets (13723 down to 7) — is what
over-aggressive state merging looks like, and 6.2 is exactly such a merge. That makes
it a plausible cause, but the only way to settle it is to apply the fix and re-measure.

The three `CoreRebecaModelsTest` errors are separate: the `NullPointerException` in
`ActivationRecord.hashCode` and the `StackOverflowError` in the depth-first search are
untouched by any of the above.

## 8. Open questions

- In `TimedRebecaActorState.receiveMessage`, messages arriving at the same instant are
  ordered by *descending* name. Any deterministic order is sound, so this may well be
  intentional, but it reads like a comparison that could have been meant the other way.
- `SendMessageRule` lacking `@Component` while the other ten statement-level rules
  have it: deliberate because `TimedRebecaSendMessageSOSRule` extends it, or an
  oversight?
- Should `ActorScope` name lookup tolerate a missing environment instead of raising
  `NullPointerException`?

## 9. Still uncovered

Known gaps, in rough order of value:

- `TimedRebecaActorState`: `memoizedClone`, `isEnable`, `createNewActorState`.
- `TimedRebecaSystemState` and `TimedActorScope` have no direct tests.
- `ActorReceivingBucket` is exercised only through the classes above it.
- The timed variants of the network delivery rules (`TimedRebecaNetworkLevelDeliverMessage`,
  `TimedRebecaFTTSNetworkLevelDeliverMessage`).

## 10. Log

| Date | Change |
|---|---|
| 2026-08-30 | Baseline recorded (44 tests, 3 failures, 3 errors). 53 tests added across five classes; suite at 97. Three defects found and reproduced. `src/main` unchanged. |
