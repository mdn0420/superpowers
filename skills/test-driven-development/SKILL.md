---
name: test-driven-development
description: Use when implementing any feature or bugfix, before writing implementation code
---

# Test-Driven Development (TDD)

## Overview

Write the test first. Watch it fail. Write minimal code to pass.

**Core principle:** If you didn't watch the test fail, you don't know if it tests the right thing. **Violating the letter of the rules is violating the spirit of the rules.**

## The Iron Law

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

Applies to features, bugfixes, refactors, and behavior changes. Throwaway prototypes, generated code, and config files are exceptions your human partner grants — ask, don't assume.

Already wrote production code for this change? Tell your human partner and get their go-ahead to discard it, then start from the test. Put the old version where you cannot read it — stash it, close the file — and implement fresh from what the test demands. Code you can still see is code you will adapt, and adapting it is testing after.

## Seams: Decide Where Tests Go

A **seam** is the public boundary you test at: the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

A seam is a boundary, not an inventory. Listing every exported name is not a seam list — that is the coverage mandate wearing a new word. Name the few boundaries where behavior worth pinning crosses, and say out loud which ones you are choosing not to test.

**Before writing any test, write down the seams under test and confirm them with your human partner.** Ask: "What's the public interface here, and which seams should we test?" No test is written at an unconfirmed seam. You cannot test everything, and testing everything evenly drains effort away from the code that matters — agreeing the seams up front is what lands your testing on critical paths and complex logic instead of on every symbol you happened to touch.

**The bar is behavior, not coverage.** A test earns its place by pinning something a user or caller depends on. A new function is not a reason to write a test; a getter, a pass-through, or a constructor that only assigns earns one when it validates, normalizes, defaults, derives, or causes a side effect — otherwise assert the first consumer-visible result that depends on it.

## Red-Green-Refactor

Inside the agreed seams: one test, one implementation, then repeat. Each test is a tracer bullet that responds to what the last cycle taught you.

### RED - Write One Failing Test

<Good>
```typescript
test('retries failed operations 3 times', async () => {
  let attempts = 0;
  const operation = () => {
    if (++attempts < 3) throw new Error('fail');
    return 'success';
  };

  expect(await retryOperation(operation)).toBe('success');
  expect(attempts).toBe(3);
});
```
Clear name, tests real behavior at the seam, one thing
</Good>

<Bad>
```typescript
test('retry works', async () => {
  const mock = jest.fn()
    .mockRejectedValueOnce(new Error())
    .mockRejectedValueOnce(new Error())
    .mockResolvedValueOnce('success');
  await retryOperation(mock);
  expect(mock).toHaveBeenCalledTimes(3);
});
```
Vague name, asserts on the mock instead of the code
</Bad>

**Requirements:** one behavior ("and" in the name? split it), a name that describes that behavior, real code, and an expected value derived independently of the code under test.

### Verify RED - Watch It Fail

**MANDATORY. Never skip.** Run the project's test command against the new test file. Confirm it fails, and that the failure message is the one you predicted.

| What you see | What it means | Do |
|---|---|---|
| Test passes | You're testing behavior that already exists | Fix the test to pin the new behavior — or drop it, it pins nothing |
| Test errors (import, syntax, setup) | An error is not a real failure | Fix the error, re-run until it fails on the assertion |
| Fails for a different reason than predicted | The test isn't measuring what you think | Back to RED. Fix and re-run until the message matches |

### GREEN - Minimal Code

<Good>
```typescript
async function retryOperation<T>(fn: () => Promise<T>): Promise<T> {
  for (let i = 0; i < 2; i++) {
    try { return await fn(); } catch { /* retry */ }
  }
  return fn();
}
```
Just enough to pass
</Good>

<Bad>
```typescript
async function retryOperation<T>(
  fn: () => Promise<T>,
  options?: { maxRetries?: number; backoff?: 'linear' | 'exponential' }
): Promise<T> { /* YAGNI */ }
```
Over-engineered for tests that don't exist yet
</Bad>

Write only what this test demands. Leave other code alone.

### Verify GREEN - Watch It Pass

**MANDATORY.** Run the project's test command.

Confirm this test passes, every other test still passes, and the output is pristine — no errors, no warnings. Test still fails? Fix the code, not the test. Other tests fail? Fix them now.

### REFACTOR - Clean Up

Green is what buys you this step. Remove duplication, improve names, extract helpers — the tests you just wrote are what make it safe. Stay green, add no behavior. Then write the next failing test.

## Anti-Patterns

- **Horizontal slicing** — writing all the tests first, then all the implementation. Bulk tests verify *imagined* behavior: you pin the shape of things instead of user-facing behavior, and you commit to test structure before you understand the implementation. Work in **vertical slices**: one test, one implementation, repeat.
- **Tautological** — the assertion recomputes the expected value the way the code does, so it passes by construction and can never disagree with the code. Expected values come from an independent source of truth: a known-good literal, a worked example, the spec.
- **Implementation-coupled** — mocks internal collaborators, tests private methods, or verifies through a side channel (querying the database instead of using the interface). The tell: the test breaks when you refactor but behavior hasn't changed.

## Good Tests

When writing or changing any test, read [writing-good-tests.md](writing-good-tests.md) for the rules that keep tests honest: naming the break each test catches, deriving expectations independently, asserting on real behavior rather than on mocks, keeping test-only code out of production classes, and running the mutation check before you finish.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "I'll test after" | Tests written after pass immediately — which proves nothing. You never watched it fail, so you never proved it can catch the bug. Test-first forces that failure. |
| "Tests after achieve the same goals (spirit not ritual)" | Tests-after answer "what does this do?"; tests-first answer "what should this do?" Tests written after are biased by the code you already wrote — you verify the cases you remembered, not the ones you'd have discovered. |
| "Deleting X hours of work is wasteful" | Sunk cost — that time is spent either way. The real choice: rewrite with TDD (high confidence) vs. bolt tests onto code you can't trust (low confidence, likely bugs). |
| "Restarting from the test doesn't fit the deadline" | The deadline is almost always on shipping, not on merging. Demo or ship from the branch you already have, and restart from the test for what actually lands. |
| "I'll write the tests now and implement after" | Horizontal slicing. You'd be pinning imagined behavior before you understand the implementation. One test, then its implementation. |
| "This function is new, so it needs a test" | Coverage is not the bar. The bar is a behavior at an agreed seam that a caller depends on. |
| "Need to explore first" | Fine. Explore, then throw the exploration away and start from the test. |
| "I'll keep the old version alongside for reference" | You will adapt it, and adapting is testing after. Stash it where you cannot read it. |
| "It's a named seam, so it needs a test even if it's a pass-through" | Naming a member did not make it a boundary. Pin the behavior that depends on it. |

Any of these means: stop, and restart this cycle from the test.

## Verification Checklist

Before marking work complete:

- [ ] Seams confirmed with your human partner before any test was written
- [ ] Every agreed seam has a test
- [ ] Every test pins a behavior a user or caller depends on — none exists because a symbol is public
- [ ] One test, one implementation, per cycle — no batch of tests written ahead of the code
- [ ] Watched each test fail before implementing
- [ ] Each test failed for the predicted reason (behavior missing, not a typo)
- [ ] Wrote minimal code to pass each test
- [ ] Every expected value derived independently of the code under test
- [ ] All tests pass, output pristine (no errors, warnings)
- [ ] Tests use real code (mocks only at system boundaries)
- [ ] Edge cases and error paths covered at the agreed seams

Can't check all boxes? You skipped TDD. Start over.

## When Stuck

| Problem | Solution |
|---------|----------|
| Don't know how to test | Write wished-for API. Write assertion first. Ask your human partner. |
| Don't know where the seam is | Ask your human partner. Unconfirmed seam, no test. |
| Test too complicated | Design too complicated. Simplify interface. |
| Must mock everything | Code too coupled. Use dependency injection. |
| Test setup huge | Extract helpers. Still complex? Simplify design. |

## Debugging Integration

Bug found? Write one failing test at the seam that reproduces it, then follow the cycle. The test proves the fix and prevents the regression. Never fix bugs without a test.
