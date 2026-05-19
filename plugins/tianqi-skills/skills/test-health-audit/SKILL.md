---
name: test-health-audit
description: "Use when reviewing test suites to find low-value tests that lock down implementation details instead of executable specifications: over-mocking, trivial accessors, redundant coverage, private-internal testing, and AI-generated coverage filler."
---

# Test Health Audit

Audit tests for refactorability. Preserve tests that specify externally observable behavior; remove or weaken tests that freeze implementation choreography.

Core invariant: **test public contracts and high-value invariants, not private call paths.**

## Trigger

Use for: "audit tests", "review test health", "find low-value tests", "remove brittle tests", "tests block refactoring", "coverage is high but confidence is low", or before AI-assisted refactoring.

Default mode: report findings only. Modify tests only when explicitly asked to remediate.

## Mental Model

| Keep / strengthen | Delete / simplify |
|---|---|
| Tests of user-visible behavior, API contracts, state transitions, persistence effects, serialization, errors, security, billing, audit, retries, routing contracts | Tests of helper call order, private methods, constructor field assignment, getters/setters, one-to-one mappers, generated coverage filler, mock wiring with no outcome assertion |

Decision rule: if the production implementation can be refactored without changing observable behavior, tests should still pass.

## Workflow

### 1. Inventory tests

Find candidate test files with:

```text
**/*Test*.*  **/*Tests*.*  **/*Spec*.*  **/*_test.*  **/test_*.*  **/*.test.*  **/*.spec.*
```

Filter to test source files in the repo's languages. Ignore snapshots, golden files, fixtures, generated artifacts, and vendored dependencies unless directly referenced by tests.

### 2. Classify test tier

| Tier | Signals | Use in audit |
|---|---|---|
| E2E / integration | real app/server/process, HTTP client, browser, real DB/container, filesystem/process boundary | high-level behavior evidence |
| Contract / API | public endpoint, public SDK/API, serialization/deserialization, command handler input/output | high-level behavior evidence |
| Unit | mocks/stubs/fakes, direct class/function calls, isolated dependencies | primary smell candidates |

When unsure, prefer the higher tier only if the test asserts behavior through a public boundary.

### 3. Build coverage map

Purpose: know which behaviors are already protected by high-level tests before judging unit tests as redundant.

Build a sparse, audit-scoped map. Do not model the whole system exhaustively; map only behaviors relevant to suspicious unit tests and deletion/replacement decisions.

Schema:

```text
coverage_map: Map<observable_behavior, evidence[]>
evidence := { tier, file, test, assertions[] }
observable_behavior := endpoint contract | public method contract | user flow | durable side effect | error/resilience invariant
```

Example:

```js
coverage_map = {
  "POST /login with valid credentials => 200 + token": [
    { tier: "contract", file: "tests/auth-api.test.ts", test: "returns token", assertions: ["status=200", "body.token"] }
  ],
  "createOrder(items) => saved order with computed total": [
    { tier: "integration", file: "tests/order-flow.test.ts", test: "creates order", assertions: ["total", "status", "persisted row"] }
  ],
  "checkout with invalid payment => rejected with user-safe error": [
    { tier: "contract", file: "tests/checkout.test.ts", test: "rejects invalid payment", assertions: ["4xx", "error code/message"] }
  ]
}
```

Use it like this: if `coverage_map` already proves "createOrder computes total", then a unit test that only verifies `taxCalculator.calculate()` was called is likely implementation-detail coverage unless it asserts a unique edge case not covered elsewhere.

### 4. Analyze unit tests

Before flagging an interaction assertion, inspect enough production code to classify the collaborator:

```text
owned helper/internal adapter | domain dependency | external boundary port | policy/safety gate | durable side-effect sink
```

For each unit test method/case, record:

```text
{ file, test, smells[], severity, action, evidence, covered_by?, replacement_needed? }
```

Prefer concrete evidence: assertion lines, mock count, private access mechanism, matching high-level coverage entry.

Large suites: partition by test project/package/directory. Build the high-level coverage map first, analyze unit-test partitions independently, then merge findings. Prefer concrete findings over exhaustive enumeration.

## Smell Detectors

| Smell | Strong signals | Default action |
|---|---|---|
| Over-mocking internals | >3 mocks; only verifies collaborator calls; verifies call order; mocks helpers owned by SUT; mocks implementation adapters instead of boundary ports | delete if verify-only internal choreography; remove interaction assertions if outcome assertions remain |
| Trivial accessor / construction | getter/setter round trip; constructor assigns field/property; 1:1 mapper copy with no transformation/validation; enum/string constant passthrough | delete unless guarding a real compatibility contract |
| Redundant with higher-level | same behavior exists in coverage_map; unit adds no edge case, invariant, or faster failure signal | delete or mark redundant |
| Private/internal testing | reflection; friend/internal visibility only for tests; `_private` access; non-public method call; test-only hooks | delete or replace with public-boundary test |
| AI-generated low-value | `[GeneratedCode]`, `// Generated by`, coverage-agent comments, repetitive mock setup, asserts only non-null/default, duplicates existing scenario | delete or rewrite around observable behavior |

## Interaction Assertion Triage

Treat these as interaction assertions:

```text
Moq Verify/VerifyAll, NSubstitute Received/DidNotReceive, Mockito verify/verifyNoInteractions,
Jest toHaveBeenCalled*, Python assert_called*, testify AssertCalled/AssertNumberOfCalls,
custom helpers containing Verify/Received/AssertCalled.
```

| Category | Detection | Action |
|---|---|---|
| Verify-only | interaction assertion is the only assertion; no output/state/error assertion | delete only if internal choreography, not contract evidence |
| Verify + outcome | meaningful output/state assertions already prove behavior | remove interaction assertion only |
| Verify as contract | interaction is the observable requirement or cannot be inferred from result | keep |

Keep interaction assertions for critical boundary contracts:

```text
auth/security checks, payment/billing calls, audit events, external API payloads,
retry/rate-limit counts, dispatch/routing decisions, no-send/no-write safety gates.
```

Heuristic: internal collaborator call = suspicious; external side effect or policy boundary = likely contract.

## Assertion Strength

After removing interaction assertions, classify remaining assertions:

| Remaining assertion | Strength | Action |
|---|---|---|
| checks meaningful status/value/error/state/persisted effect | strong | keep |
| non-null plus count/content/value checks | strong | keep |
| only non-null/default/instance type for complex result | weak | strengthen or delete |
| no assertions | none | delete |

Strengthen only if the test still covers a unique behavior. Otherwise delete.

## Severity

| Severity | Criteria |
|---|---|
| High | actively blocks valid refactoring: verify-only internal choreography, private method tests, exact internal sequence, 5+ mocks, no outcome assertions |
| Medium | low confidence value: trivial accessors, constructor assignment, redundant unit around already-covered behavior |
| Low | borderline: partial redundancy, weak assertion but some unique edge case, possible contract ambiguity |

## Remediation Algorithm

If asked to fix findings:

1. Delete verify-only tests only when the verified interaction is internal choreography, not an observable contract, and either behavior is covered elsewhere or the test specifies no real behavior.
2. Remove redundant interaction assertions from tests with strong outcome assertions.
3. Re-run or inspect remaining assertions; strengthen weak assertions only for unique behavior.
4. Delete trivial accessor/private-internal/redundant tests.
5. Run the relevant test suite after each coherent batch.
6. If deleting the only coverage for real behavior, replace with a public-boundary or outcome-focused test instead of deleting outright.

If the interaction is the only evidence of a real external side effect, safety gate, audit/payment/security action, retry/routing policy, or no-send/no-write rule, keep it or replace it with a public-boundary/outcome-focused test.

Never reduce coverage of security, billing, audit, external API, retry/rate-limit, or safety-gate behavior without an equivalent public/outcome assertion.

## Report Contract

Print markdown. Be compact and actionable.

```markdown
# Test Health Audit Report

## Summary
- Test files scanned: X
- Test methods analyzed: Y
- Findings: Z
- Safe delete candidates: N
- Remove-interaction-only candidates: M
- Replacement-needed candidates: R

## Coverage Map
| Behavior | Existing high-level evidence |
|---|---|
| POST /login with valid credentials => 200 + token | tests/auth-api.test.ts:returns token |

## Findings
| File | Test | Smell | Severity | Action | Evidence | Covered by |
|---|---|---|---|---|---|---|
| tests/order-service.test.ts | CreateOrder_CallsTaxCalculator | Over-mocking | High | Remove interaction assertion | final total already asserted; helper call is choreography | tests/order-flow.test.ts:creates order |
| tests/repo.test.ts | Save_OnlyVerifiesCall | Verify-only | High | Delete test | no output/state assertion | none; no real behavior specified |

## Recommendations
- Delete: ...
- Remove interaction assertions only: ...
- Strengthen or replace: ...
- Keep as contract: ...
```

Do not estimate "coverage percentage saved"; report concrete candidates and evidence instead.

## Common Failure Modes

| Failure | Correction |
|---|---|
| Flagging all mocks | mocks of external/boundary ports may be contract tests; mocks of owned helpers are suspicious |
| Deleting only coverage for real behavior | replace with public-boundary/outcome test |
| Treating retry/payment/audit verifies as over-mocking | keep when interaction is the requirement |
| Counting tests instead of mock density/assertion quality | prioritize behavior value per test |
| Removing Verify then leaving weak assertions | strengthen unique behavior or delete |
| Missing framework aliases | search for Verify, Received, assert_called, toHaveBeenCalled, verify(, AssertCalled |
