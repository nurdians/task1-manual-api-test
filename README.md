# QA Take-Home Test — Task 1: Manual API Test

Deliverables for **Task 1 (Manual API Test)** of the *Take Home Test – Junior to Medior QA Engineer*.

## Contents

| File | Purpose |
|---|---|
| `test-cases.md` | Test case document — 15 cases (TC-POST-01…04, TC-GET-01…06, TC-DEL-00…04) with expected results derived from the official spec |
| `petstore-task1.postman_collection.json` | Postman collection implementing the test cases (importable in the Postman GUI, runnable with the Postman CLI) |
| `report.md` | Execution report — actual outcomes per test case, statistics, findings, evidence index |
| `bug-report.md` | Bugs and documentation mismatches found (8 bugs + 3 documentation issues) |
| `evidence/collection-run.log` | Raw console output of the collection run (all 50 assertions) |
| `evidence/verification.log` | Post-run verification probes (data-integrity check, id-precision investigation, stability checks) |

## Requirements

- Postman app (GUI) **or** Postman CLI (`postman` — v1.73.0 was used here)
- Internet access to `https://petstore.swagger.io` (public demo, no auth needed)

## How to run

**Postman GUI**

1. Import `petstore-task1.postman_collection.json` (Import → Files).
2. Open **QA Take-Home Task 1 - Petstore API Test** → **Run** (Collection Runner).
3. Keep the order as-is (TC-POST-* → TC-GET-* → TC-DEL-*), iterations = 1 → **Run**.

**Postman CLI**

```bash
postman collection run petstore-task1.postman_collection.json --no-report-events --timeout-request 20000
```

> The exit code is currently **non-zero** because 8 assertions fail — that is the expected result: the failures are the documented-vs-actual mismatches recorded in `report.md` / `bug-report.md`. Login is **not** required for running a local collection (only the `--output` artifact flag would need it).

## Target & test oracle

- API base URL: `https://petstore.swagger.io/v2`
- Specification: `https://petstore.swagger.io/v2/swagger.json` (retrieved 12 Nov 2026)

## Note on test data

The target is a shared public demo. All pets created during test execution were deleted afterwards; pre-existing invalid records (e.g. `id=98766` without `name`) were left untouched as bug evidence.
