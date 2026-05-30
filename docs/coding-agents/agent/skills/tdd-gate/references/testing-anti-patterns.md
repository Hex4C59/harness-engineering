# Testing Anti-Patterns

Read this when a TDD cycle involves mocks, test-only hooks, broad fixtures, flaky tests, or uncertainty about whether the test proves real behavior.

## Testing The Mock

Red flag:

```text
The test mainly verifies that a mock method was called.
```

Better:

- Assert the observable behavior produced by the unit under test.
- Mock only the external boundary that cannot reasonably run in the test.
- Keep one integration or contract test for the real boundary when risk is meaningful.

## Implementation-Coupled Assertions

Red flag:

```text
A harmless refactor breaks the test even though behavior is unchanged.
```

Better:

- Test public outputs, persisted state, rendered result, emitted event, or documented error.
- Avoid asserting private method calls, internal ordering, or intermediate variables unless that is the contract.

## Test-Only Production APIs

Red flag:

```text
Production code gains methods or flags only so tests can reach internals.
```

Better:

- Prefer testing through existing public behavior.
- If a seam is needed, make it a real dependency boundary that also improves design.
- Do not add test-only conditionals to production paths.

## Weak RED

Red flag:

```text
The test fails for setup reasons, or would pass for many wrong implementations.
```

Better:

- Read the failure message before implementing.
- Strengthen assertions with specific expected values and edge cases.
- Make the test fail because the target behavior is missing, not because the test environment is broken.

## Over-Mocking Existing Code

Red flag:

```text
The test replaces most collaborators and no longer exercises the real behavior.
```

Better:

- Use real collaborators when they are fast, deterministic, and local.
- Mock network, time, randomness, filesystem, external services, and expensive dependencies.
- Add a higher-level test when unit-level mocks hide important integration risk.

## Changing The Test To Pass

Red flag:

```text
After implementation fails, the test expectation is weakened or rewritten to match the implementation.
```

Better:

- Re-check the requirement, Spec, bug report, or user request.
- Only change the test when the original expectation was wrong, and say why.
- If the implementation reveals a requirement conflict, stop and ask for the smallest needed decision.
