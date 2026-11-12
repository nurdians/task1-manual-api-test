# Task 1 — Manual API Test: Test Report

**Take Home Test – Junior to Medior QA Engineer**

## 1. Execution Summary

| Item | Value |
|---|---|
| Date executed | 12 Nov 2026 |
| Target | Swagger Petstore — `https://petstore.swagger.io/v2` |
| Specification (test oracle) | `https://petstore.swagger.io/v2/swagger.json` (retrieved 12 Nov 2026) |
| Tool | Postman collection `petstore-task1.postman_collection.json`, executed with Postman CLI **v1.73.0** |
| Command | `postman collection run petstore-task1.postman_collection.json --no-report-events --timeout-request 20000` |
| Exit code | `1` — non-zero because 8 assertions failed (failures = spec-vs-actual mismatches, see §2) |
| Raw evidence | `evidence/collection-run.log` (full console output), `evidence/verification.log` (post-run verification probes) |
| Test cases | `test-cases.md` — request names in the collection are identical to the TC IDs |

**Run statistics (from the run summary):**

| Metric | Value |
|---|---|
| Requests executed | 15 (0 transport/request errors) |
| Assertions | 50 executed — **42 passed, 8 failed** |
| Test scripts / pre-request scripts | 15 / 2 — 0 script errors |
| Total run duration | 7.2 s |
| Response time | avg 462 ms (min 258 ms, max 2.5 s) |
| Data received | 102.62 KB |

**Test-case outcome:** 7 passed (2 of them with findings), 6 failed, 2 partially failed.

## 2. Result Overview

| TC | Expected (per spec) | Actual | Result | Finding |
|---|---|---|---|---|
| TC-POST-01 | `200` + created pet JSON | `200` (539 B, 2.5 s), numeric `id` > 0, submitted data echoed | ✅ PASS (5/5) | DOC-01 |
| TC-POST-02 | `405 Invalid input` | **`200`** — pet stored **without** the required `name` | ❌ FAIL (1/2) | **BUG-01** |
| TC-POST-03 | `405 Invalid input` | **`500`** `{"code":500,"type":"unknown","message":"something bad happened"}` | ❌ FAIL (1/2) | **BUG-02** |
| TC-POST-04 | `405 Invalid input` | **`200`** — invalid `status` accepted and persisted | ❌ FAIL (1/2) | **BUG-03** |
| TC-GET-01 | `200`, every item has `id`, `name`, `status=available` | `200` (38.92 KB) but record `id=98766` has **no `name`** | ⚠ PARTIAL (5/6) | BUG-01 (data impact) |
| TC-GET-02 | `200`, every item `status=pending` | `200` (4.51 KB), all assertions green | ✅ PASS (6/6) | — |
| TC-GET-03 | `200`, every item `status=sold` | `200` (20.56 KB), all assertions green | ✅ PASS (6/6) | — |
| TC-GET-04 | `400` (`status` is required) | **`200`** — run: 328 B body; 9 re-checks: `[]` | ❌ FAIL (1/2) | **BUG-04** |
| TC-GET-05 | `400 Invalid status value` | **`200`** with results (480 B) | ❌ FAIL (1/2) | **BUG-05** |
| TC-GET-06 | `200`, items ∈ {available, sold}, **both** present | `200` but **only the first value applied** (253 items = available-only) | ⚠ PARTIAL (5/6) | **BUG-06** |
| TC-DEL-00 | `200`, store `deletePetId` | `200`, stored id `9223372036854565121` | ✅ PASS (3/3) | — |
| TC-DEL-01 | `200` pet deleted | `200` — but issued for id `…65000`, **not** the created `…65121` | ⚠ PASS w/ finding (2/2) | **BUG-08** |
| TC-DEL-02 | `404` pet is gone | `404` (292 B) — however the created pet still existed after the run | ⚠ PASS w/ finding (2/2) | **BUG-08** |
| TC-DEL-03 | `404 Pet not found` | `404` (292 B) | ✅ PASS (2/2) | — |
| TC-DEL-04 | `400 Invalid ID supplied` | **`404`** (433 B) | ❌ FAIL (1/2) | **BUG-07** |

> Assertion counts: TC-POST-01 5, TC-GET-01/02/03/06 6 each, TC-DEL-00 3, all other cases 2 each = 50 total; the 8 failures are the status-code/data assertions listed above.

## 3. Detailed Findings by Endpoint

### 3.1 `POST /pet`

**TC-POST-01 (positive) — PASS.** `200 OK`, 539 B, 2.5 s; response contains a new numeric `id` > 0 and echoes `name`, `category`, `photoUrls`, `status`. The spec, however, documents **no success response** for this endpoint (only `405`) — filed as DOC-01.

**TC-POST-02 — expected `405`, actual `200`.** The API accepts a payload that violates the required `name` field and stores it. Response captured during verification (the run's own response was `200` / 435 B with the same shape):

```json
{"id":9223372036854565157,"photoUrls":["https://example.com/pet/evidence-no-name.jpg"],"tags":[],"status":"available"}
```

The `name` key is entirely absent. Corroborating evidence of real-world impact: the shared database already contains a nameless pet — `GET /pet/98766` → `200`:

```json
{"id":98766,"category":{"id":1,"name":"Dogs"},"photoUrls":["string"],"tags":[{"id":1,"name":"friendly"}],"status":"available"}
```

(no `name`), which is exactly the record that failed TC-GET-01's "every result has id and name" assertion. → **BUG-01**

**TC-POST-03 — expected `405`, actual `500`.** Sending `"id": "not-a-number"` (string vs documented int64) triggers an unhandled server error:

```json
{"code":500,"type":"unknown","message":"something bad happened"}
```

→ **BUG-02**

**TC-POST-04 — expected `405`, actual `200`.** The enum violation is accepted and persisted:

```json
{"id":9223372036854565159,"name":"QA Evidence Bad Status","photoUrls":["https://example.com/pet/evidence-bad-status.jpg"],"tags":[],"status":"not-a-valid-status"}
```

The record is retrievable by `GET /pet/{id}` and by `findByStatus?status=not-a-valid-status`, i.e. invalid data lives in the store and is only discoverable by querying the invalid value itself. A pre-existing record (`id=9223372036854565001`) carries `status:"string"` — proof that this pollution is ongoing in the shared demo. → **BUG-03**

### 3.2 `GET /pet/findByStatus`

**TC-GET-01/02/03 (positive) — filtering works.** All three return `200` with non-empty arrays and correct `status` values (`available` 38.92 KB, `pending` 4.51 KB, `sold` 20.56 KB). TC-GET-01 still failed one assertion because the result set contains the nameless record `id=98766` (BUG-01 impact).

**TC-GET-04 — expected `400`, actual `200`.** The required `status` parameter is not enforced. During the run the response was `200` with a 328 B body; nine re-checks across two verification sessions consistently return `200 []` (2 B). The status mismatch is stable; the body-size difference between run and re-check is attributed to the shared demo environment (§5). → **BUG-04**

**TC-GET-05 — expected `400 Invalid status value`, actual `200`.** `?status=not-a-valid-status` returns `200` with an array of pets that carry that invalid status (480 B during the run — the shared DB accumulates such records from concurrent testers) instead of rejecting the value. → **BUG-05**

**TC-GET-06 — expected OR-semantics for multi-value `status`, actual = first value only.**

| Query | Items | Size |
|---|---|---|
| `status=available` | 253 | 38.92 KB |
| `status=sold` | 94 | 20.56 KB |
| `status=available&status=sold` | **253** | **38.92 KB — byte-identical to `status=available`** |

The response for the multi-value query is byte-identical to the single `available` query: all `sold` pets (94) are silently dropped, although the spec declares `collectionFormat: multi` (OR semantics). → **BUG-06**

### 3.3 `DELETE /pet/{petId}`

**TC-DEL-00 (setup) — PASS.** Created the target pet; exact server id = `9223372036854565121` (19 digits).

**TC-DEL-01 / TC-DEL-02 — assertions passed, but the test's intent was NOT achieved (BUG-08).** During the run the sequence was:

1. `TC-DEL-00` created pet **`9223372036854565121`**.
2. `TC-DEL-01` issued `DELETE /pet/9223372036854565000` → `200`; `TC-DEL-02` repeated it → `404`.
3. **Post-run check: `GET /pet/9223372036854565121` → `200` — the pet created by the test still existed.** The collection never deleted the pet it created.

Root cause: the API issues 19-digit ids near `int64` max (≈ 9.22 × 10¹⁸). JavaScript's `Number.MAX_SAFE_INTEGER` is 9007199254740991 (≈ 9.0 × 10¹⁵), so any JS-based client (Postman runner, Swagger UI, Node apps) rounds the id — `…65121` became `9223372036854565000` in the request URL. Because the API reliably returns `404` for ids that do not exist (verified 5× on `987654321098`), the `200` of TC-DEL-01 most likely removed a **different, pre-existing demo record** with that exact id.

Control probe proves the endpoint itself works when the exact id is used (Python client, no precision loss):

```
POST /pet            -> 200, exact id 9223372036854565253
DELETE /pet/<exact>  -> 200 {"code":200,"type":"unknown","message":"9223372036854565253"}
GET  /pet/<exact>    -> 404 {"code":1,"type":"error","message":"Pet not found"}
```

→ **BUG-08** (API design: id range unsafe for JSON/JS consumers).

**TC-DEL-03 — PASS.** `DELETE /pet/999999999` → `404 Pet not found`, as documented.

**TC-DEL-04 — expected `400 Invalid ID supplied`, actual `404`.** A non-numeric id (`/pet/abc`) is reported as *not found* instead of *invalid id*, conflating two different error conditions. → **BUG-07**

## 4. Bugs & Documentation Mismatches Found

Full details, reproduction steps and evidence: **`bug-report.md`**.

| ID | Severity | Endpoint | Title |
|---|---|---|---|
| BUG-01 | High | `POST /pet` | Required `name` not validated — invalid records stored (200) |
| BUG-02 | High | `POST /pet` | Wrong type for `id` causes unhandled `500` instead of a validation error |
| BUG-03 | Medium | `POST /pet` | `status` enum not validated on create — invalid values persisted |
| BUG-04 | Medium | `GET /pet/findByStatus` | Required `status` parameter not enforced — `200 []` instead of `400` |
| BUG-05 | Medium | `GET /pet/findByStatus` | Invalid `status` value not rejected — `200` instead of `400` |
| BUG-06 | High | `GET /pet/findByStatus` | Multi-value `status` query silently ignores every value after the first |
| BUG-07 | Low | `DELETE /pet/{petId}` | Non-numeric `petId` returns `404` instead of documented `400` |
| BUG-08 | High | all (ID handling) | 19-digit ids exceed `Number.MAX_SAFE_INTEGER` — JS clients operate on the wrong record |
| DOC-01 | Medium | `POST /pet` | No success response documented (only `405`) |
| DOC-02 | Medium | `DELETE /pet/{petId}` | No success response documented (only `400`/`404`) |
| DOC-03 | Low | `POST /pet` | Invalid input documented as `405` (Method Not Allowed) — semantically wrong and never observed in practice |

**Totals:** 8 bugs (4 High, 3 Medium, 1 Low) + 3 documentation issues (2 Medium, 1 Low).

## 5. Environment Notes & Limitations

- **Shared public demo.** `petstore.swagger.io` is used by many visitors (including other candidates running this same take-home). Data can change between the run and a re-check — e.g. TC-GET-04's body was 328 B during the run and `[]` on all 9 re-checks; the status-code mismatch itself was stable. Response sizes for `findByStatus` also grow over time as testers create pets.
- **Expected results are spec-derived.** Where the spec documents no success response (POST, DELETE), the tests assert the sensible `200` and flag the gap (DOC-01/DOC-02) instead of failing silently.
- **Postman CLI without login.** Running a local collection does not require authentication; the `--output` flag (YAML run artifacts) does, so the raw console output was captured to `evidence/collection-run.log` instead. The login notices in that log are cosmetic — the run completed normally.
- **Test-data cleanup.** All 7 pets created during testing were deleted afterwards: `…65118` (TC-POST-01), `…65119` (TC-POST-02), `…65120` (TC-POST-04), `…65121` (TC-DEL-00), `…65157`/`…65159` (verification evidence), `…65253` (control probe). Pre-existing invalid records (`id=98766` without `name`, `id=…65001` with `status:"string"`) were **left untouched as bug evidence**.
- **Response-time threshold** was set to < 3000 ms for a public demo; every timing assertion passed (max observed 2.5 s on TC-POST-01).

## 6. Evidence Index

| File | Content |
|---|---|
| `evidence/collection-run.log` | Raw Postman CLI output of the full run: per-request status/size/time, all 50 assertion results, failure details, run summary |
| `evidence/verification.log` | Post-run probes: deletion verification, id-precision investigation, control DELETE cycle, stability checks (5×/6× repeats) |
| `test-cases.md` | The 15 test cases with spec-derived expected results |
| `petstore-task1.postman_collection.json` | The executed Postman collection |
| `bug-report.md` | Bug and documentation-mismatch reports with reproduction steps |
