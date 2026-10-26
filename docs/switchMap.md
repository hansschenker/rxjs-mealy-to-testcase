# switchMap: Mealy → STT → Testcases

## Behavioral derivation

`switchMap` can be analyzed as a stateful transducer whose behavior depends on:

- source lifecycle,
- presence or absence of an active inner subscription,
- inner lifecycle,
- downstream lifecycle.

The useful event alphabet is:

```text
source.next(x)
source.error(e)
source.complete
inner.next(y)
inner.error(e)
inner.complete
downstream.subscribe
downstream.unsubscribe
```

Subscription management of the inner observable belongs to the operator's Action Policy.

## Minimum test obligations

The following 11 rows capture the reachable behaviorally distinct cases that should be represented by tests:

1. source `next` while no inner is active
2. source `next` while an inner is active
3. source `complete` while no inner is active
4. source `complete` while an inner is active
5. inner `next` while inner active
6. inner `complete` while source active
7. inner `complete` while source already completed
8. source `error`
9. inner `error`
10. downstream `unsubscribe` while inner active
11. downstream `unsubscribe` while no inner is active

## Test-name grammar

```text
Given <state>
When  <event>
Then  <action / transition>
```

maps naturally to:

```ts
it('should <action> when <event> [while <state>]');
```

## Minimum testcase names

```ts
describe('switchMap behavioral state table', () => {
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
});
```

## Important qualification

These 11 cases are **semantic test obligations**, not necessarily the complete RxJS implementation test suite.

One obligation can require several concrete tests for:

- synchronous inner observables,
- reentrant emissions,
- exact subscription frames,
- teardown ordering,
- scheduler-sensitive behavior,
- projection errors,
- deprecated/compatibility behavior if relevant.

The state table provides the behavioral coverage baseline.
