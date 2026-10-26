# RxJS Mealy to Testcase

A small methodology for deriving RxJS operator tests from a Mealy-machine behavior model.

## Core Insight

The path from operator semantics to tests can be made systematic:

```text
Operator Behavior
    ↓
Mealy Machine
    ↓
States + Input Events
    ↓
State Transition Table (STT)
    ↓
Reachable, behaviorally distinct rows
    ↓
Minimum Test Obligations
    ↓
Concrete RxJS Testcases
```

The key rule is:

> **One behaviorally distinct STT row = one minimum test obligation.**

This does **not** mean that one STT row can only have one concrete test. Timing, reentrancy, synchronous completion, scheduler behavior, and other implementation-sensitive edge cases can refine one obligation into several concrete tests.

## Why this matters

Traditional tests are often written from examples or remembered edge cases. A Mealy-derived approach starts from the operator's complete behavioral model.

For an operator:

```text
Given current state
When input event occurs
Then transition to next state
     perform Action Policy
```

the corresponding test naming grammar becomes:

```text
it('should <action> when <event> [while <state>]');
```

This creates traceability:

```text
Behavior rule
    ↕
STT row
    ↕
Test obligation
    ↕
Concrete test
```

## Input events versus actions

When modeling an RxJS operator, distinguish external input events from actions caused by the operator.

For `switchMap`, a useful input alphabet is:

```text
Downstream lifecycle
  subscribe
  unsubscribe

Outer/source observable
  next(x)
  error(e)
  complete

Current inner observable
  next(y)
  error(e)
  complete
```

Inner subscription and inner unsubscription are normally **actions of switchMap**, not independent input events.

Example:

```text
Given active inner
When source.next(x)
Then unsubscribe current inner
     project x
     subscribe new inner
```

## Reachability, not brute-force combinations

Do not test every mathematical combination of states and events.

Use:

```text
reachable state × admissible input event
```

Then keep combinations where changing the state or event changes either:

- the state transition, or
- the Action Policy.

Unreachable combinations such as `inner.next` when no inner subscription exists are not semantic scenarios.

## switchMap example

A compact `switchMap` STT produces 11 behaviorally distinct test obligations:

| # | State / condition | Input event | Expected behavior |
|---:|---|---|---|
| 1 | source active, no inner | `source.next(x)` | project `x`, subscribe new inner |
| 2 | source active, inner active | `source.next(x)` | unsubscribe old inner, project `x`, subscribe new inner |
| 3 | source active, no inner | `source.complete` | complete downstream immediately |
| 4 | source active, inner active | `source.complete` | remember source completion, keep inner alive |
| 5 | inner active | `inner.next(y)` | forward `y` downstream |
| 6 | source active, inner active | `inner.complete` | clear active inner, wait for more source values |
| 7 | source completed, inner active | `inner.complete` | clear inner, complete downstream |
| 8 | source active | `source.error(e)` | error downstream, tear down active inner |
| 9 | inner active | `inner.error(e)` | error downstream, tear down source |
| 10 | inner active | downstream `unsubscribe` | unsubscribe source and active inner |
| 11 | no inner active | downstream `unsubscribe` | unsubscribe source |

These rows become the minimum behavioral testcase set.

## Test descriptions

```ts
it('should subscribe to a new inner observable when the source emits and no inner is active');

it('should unsubscribe from the current inner observable and subscribe to a new inner observable when the source emits while an inner is active');

it('should complete immediately when the source completes and no inner is active');

it('should wait for the active inner observable to complete when the source completes');

it('should forward values emitted by the active inner observable');

it('should clear the active inner observable and wait for another source value when the inner completes while the source is active');

it('should complete when the active inner observable completes after the source has completed');

it('should error and unsubscribe from the active inner observable when the source errors');

it('should error and unsubscribe from the source observable when the active inner observable errors');

it('should unsubscribe from both the source and the active inner observable when the downstream subscription is unsubscribed');

it('should unsubscribe from the source when the downstream subscription is unsubscribed and no inner observable is active');
```

## General derivation procedure

1. Identify the operator's input-event alphabet.
2. Identify the minimum state required to explain behavior.
3. Construct candidate `State × Event` rows.
4. Remove unreachable combinations.
5. Merge rows whose transition and Action Policy are behaviorally identical.
6. Treat every remaining row as a minimum test obligation.
7. Refine obligations into additional implementation-level tests where timing, reentrancy, scheduling, or teardown requires it.

## Principle

> **Semantics should generate the tests.**

Instead of asking, "What tests should we write for this operator?", ask:

> "What are the reachable, behaviorally distinct rows of its state-transition table?"

Once that table is complete, the minimum behavioral test suite follows directly.
