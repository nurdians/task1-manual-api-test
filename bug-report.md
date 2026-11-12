# Bug Report — Swagger Petstore v2 (QA Take-Home, Task 1)

| Field | Value |
|---|---|
| Reported | 12 Nov 2026 |
| Application | Swagger Petstore — `https://petstore.swagger.io/v2` |
| Test oracle | Official spec `https://petstore.swagger.io/v2/swagger.json` (retrieved 12 Nov 2026) |
| Found during | Task 1 — Manual API Test (`test-cases.md`, results in `report.md`) |
| Environment | Public shared demo — data is visible to and mutable by all visitors |
| Status | All issues **Open** |

**Severity scale:** **High** — incorrect/unreliable behaviour or data-integrity risk · **Medium** — spec mismatch with functional impact · **Low** — status-code/documentation mismatch without data impact.

## Summary

| ID | Severity | Endpoint | Title |
|---|---|---|---|
| BUG-01 | High | `POST /pet` | Required `name` not validated — invalid records accepted with `200` |
| BUG-02 | High | `POST /pet` | Wrong type for `id` triggers unhandled `500` instead of a validation error |
| BUG-03 | Medium | `POST /pet` | `status` enum not validated on create — invalid values persisted |
| BUG-04 | Medium | `GET /pet/findByStatus` | Required `status` parameter not enforced — `200 []` instead of `400` |
| BUG-05 | Medium | `GET /pet/findByStatus` | Invalid `status` value not rejected — `200` instead of `400` |
| BUG-06 | High | `GET /pet/findByStatus` | Multi-value `status` query silently ignores every value after the first |
| BUG-07 | Low | `DELETE /pet/{petId}` | Non-numeric `petId` returns `404` instead of documented `400` |
| BUG-08 | High | all endpoints (ID handling) | 19-digit ids exceed `Number.MAX_SAFE_INTEGER` — JS clients operate on the wrong record |

**Documentation issues** (spec vs actual behaviour):

| ID | Severity | Endpoint | Title |
|---|---|---|---|
| DOC-01 | Medium | `POST /pet` | No success response documented (only `405`); actual success = `200` |
| DOC-02 | Medium | `DELETE /pet/{petId}` | No success response documented (only `400`/`404`); actual success = `200` |
| DOC-03 | Low | `POST /pet` | Invalid input documented as `405` (Method Not Allowed) — semantically wrong, and never observed in practice |

---

## BUG-01 — Required `name` not validated on create (High)

**Endpoint:** `POST /pet`
**Spec:** `Pet.name` is listed under `required`; invalid input is documented as `405 Invalid input`.

**Steps to reproduce**

```bash
curl -s -i -X POST https://petstore.swagger.io/v2/pet \
  -H 'Content-Type: application/json' \
  -d '{"photoUrls":["https://example.com/pet/no-name.jpg"],"status":"available"}'
```

**Expected:** `405 Invalid input` (spec) — request rejected because `name` is required.
**Actual:** `200 OK`, pet created without a `name` field:

```json
{"id":9223372036854565157,"photoUrls":["https://example.com/pet/evidence-no-name.jpg"],"tags":[],"status":"available"}
```

**Evidence / impact**

- The record violates the documented schema; downstream consumers reading `pet.name` get `undefined`.
- The shared DB already contains such records, e.g. `GET /pet/98766` → `200` with no `name` key — this record caused test case TC-GET-01 to fail ("every result has id and name").
- Found by: TC-POST-02 (run response `200`, 435 B) + post-run verification.

**Recommendation:** validate required fields before persisting and return `400` (see DOC-03 on the status-code choice) with a descriptive error body.

---

## BUG-02 — Wrong type for `id` triggers unhandled `500` (High)

**Endpoint:** `POST /pet`
**Spec:** `Pet.id` is `integer/int64`; invalid input is documented as `405 Invalid input`.

**Steps to reproduce**

```bash
curl -s -i -X POST https://petstore.swagger.io/v2/pet \
  -H 'Content-Type: application/json' \
  -d '{"id":"not-a-number","name":"QA Invalid ID Type","photoUrls":["https://example.com/pet/invalid-id.jpg"],"status":"available"}'
```

**Expected:** `405 Invalid input` (spec) — a validation-level rejection.
**Actual:** `500 Internal Server Error`:

```json
{"code":500,"type":"unknown","message":"something bad happened"}
```

**Impact:** a client-side type mistake is reported as a server fault; the generic message gives no guidance, and unhandled exceptions can hide real server problems (and may leak stack traces in other configurations).

**Recommendation:** catch deserialization/type errors and map them to `400` with a field-level message.

---

## BUG-03 — `status` enum not validated on create (Medium)

**Endpoint:** `POST /pet`
**Spec:** `status` enum = `available | pending | sold`; invalid input documented as `405 Invalid input`.

**Steps to reproduce**

```bash
curl -s -i -X POST https://petstore.swagger.io/v2/pet \
  -H 'Content-Type: application/json' \
  -d '{"name":"QA Invalid Status","photoUrls":["https://example.com/pet/invalid-status.jpg"],"status":"not-a-valid-status"}'

curl -s 'https://petstore.swagger.io/v2/pet/findByStatus?status=not-a-valid-status'
```

**Expected:** `405 Invalid input` — request rejected.
**Actual:** `200 OK`, the invalid value is persisted and returned verbatim:

```json
{"id":9223372036854565159,"name":"QA Invalid Status","photoUrls":["https://example.com/pet/invalid-status.jpg"],"tags":[],"status":"not-a-valid-status"}
```

**Impact:** invalid data enters the store and can only be found by querying the invalid value itself — normal `available`/`pending`/`sold` filters can never surface it (silent data loss for consumers). Pre-existing pollution observed: record `id=9223372036854565001` carries `status:"string"`.

**Recommendation:** validate `status` against the enum on write; reject with `400`.

---

## BUG-04 — Required `status` parameter not enforced (Medium)

**Endpoint:** `GET /pet/findByStatus`
**Spec:** parameter `status` is `required: true`; documented responses are `200` and `400`.

**Steps to reproduce**

```bash
curl -s -i https://petstore.swagger.io/v2/pet/findByStatus
```

**Expected:** `400` — required parameter missing.
**Actual:** `200 OK` — during the test run a 328 B body was returned; on 9 subsequent re-checks the response was a stable `[]` (empty array).

**Impact:** clients that forget the parameter receive a misleading "success" instead of an error; behaviour of the run-time body also differed from re-checks (environment inconsistency, see report.md §5).

**Recommendation:** reject requests without `status` using `400`.

---

## BUG-05 — Invalid `status` value not rejected (Medium)

**Endpoint:** `GET /pet/findByStatus`
**Spec:** `status` items are enum-constrained; `400 Invalid status value` is documented for invalid values.

**Steps to reproduce**

```bash
curl -s -i 'https://petstore.swagger.io/v2/pet/findByStatus?status=not-a-valid-status'
```

**Expected:** `400 Invalid status value`.
**Actual:** `200 OK` with an array of pets whose `status` equals the invalid string (480 B during the run; the shared DB accumulates such records from testers — see BUG-03).

**Impact:** the documented contract is not enforced; a typo in `status` looks like a successful query with (possibly) zero results instead of an error.

**Recommendation:** validate the query value against the enum and return `400`.

---

## BUG-06 — Multi-value `status` query ignores every value after the first (High)

**Endpoint:** `GET /pet/findByStatus`
**Spec:** `status` is an array with `collectionFormat: multi` — all supplied values should be honoured (OR semantics).

**Steps to reproduce**

```bash
curl -s 'https://petstore.swagger.io/v2/pet/findByStatus?status=available'        # 253 items, 38.92 KB
curl -s 'https://petstore.swagger.io/v2/pet/findByStatus?status=sold'             #  94 items, 20.56 KB
curl -s 'https://petstore.swagger.io/v2/pet/findByStatus?status=available&status=sold'
```

**Expected:** `200` with pets whose status is `available` **or** `sold` (253 + 94 = 347 items at time of test).
**Actual:** `200` with exactly the 253 `available` items — the response is **byte-identical** to the single-value `status=available` query; all 94 `sold` pets are silently dropped.

**Impact:** any client filtering by multiple statuses receives incomplete data with no error — a silent correctness failure (worst class of API bug).

**Recommendation:** parse repeated query parameters into the array the spec declares and filter with `status IN (...)`.

---

## BUG-07 — Non-numeric `petId` returns `404` instead of `400` (Low)

**Endpoint:** `DELETE /pet/{petId}`
**Spec:** `petId` is `integer/int64`; documented responses `400 Invalid ID supplied` and `404 Pet not found`.

**Steps to reproduce**

```bash
curl -s -i -X DELETE https://petstore.swagger.io/v2/pet/abc
```

**Expected:** `400 Invalid ID supplied` — the id is not even parseable as int64.
**Actual:** `404 Not Found` (433 B) — an unparseable id is reported the same as an id that does not exist.

**Impact:** clients cannot distinguish "wrong format" from "not found"; monitoring/alerting on `400` for malformed requests will never fire.

**Recommendation:** validate the path parameter type first and return `400`; reserve `404` for parseable ids that are absent.

---

## BUG-08 — 19-digit ids exceed `Number.MAX_SAFE_INTEGER` — JS clients hit the wrong record (High)

**Endpoints:** all (ID issuance on `POST /pet`, ID consumption on `GET`/`DELETE /pet/{petId}`)

**Spec:** `Pet.id` / `petId` are `integer/int64`. `int64` values are not representable exactly in IEEE-754 doubles, which every JavaScript client (Postman runner, Swagger UI, Node.js apps) uses for JSON numbers.

**Observed during the test run**

1. `TC-DEL-00` created a pet; the exact server id was **`9223372036854565121`** (19 digits, ≈ 9.22 × 10¹⁸).
2. The Postman (JavaScript) test layer rounded it and issued `DELETE /pet/9223372036854565000` → `200`, then `404` on repeat.
3. **Post-run verification: `GET /pet/9223372036854565121` → `200` — the pet created by the test still existed.** The test deleted (or was told it deleted) a *different* id than the one it created; because the API returns `404` for ids that do not exist (verified 5× on `987654321098`), the `200` most likely removed a pre-existing demo record with that exact id.
4. Control probe with an exact id (Python client, no precision loss) shows the endpoint logic itself is fine:

```
POST /pet           -> 200, exact id 9223372036854565253
DELETE /pet/<exact> -> 200 {"code":200,"type":"unknown","message":"9223372036854565253"}
GET  /pet/<exact>   -> 404 Pet not found
```

**Numbers:** `Number.MAX_SAFE_INTEGER = 9007199254740991` (9.007 × 10¹⁵, 16 digits) vs issued ids ≈ 9.22 × 10¹⁸ (19 digits) — off by ~3 orders of magnitude.

**Impact:** from any JS client, *get-by-id* and *delete-by-id* can silently target the wrong record — data loss (wrong record deleted) or phantom "success". The same flaw affects Swagger UI's "Try it out" (the primary way this API is used).

**Recommendation:** issue ids that are JSON-safe (≤ 2⁵³ − 1, or UUIDs/strings), or serialize `id` as a string in JSON. At minimum, document the hazard.

---

## DOC-01 — `POST /pet` documents no success response (Medium)

The spec lists only `405 – Invalid input` for `POST /pet`; the actual success response is `200` with the created pet (observed on every valid create, e.g. TC-POST-01: `200`, 539 B). Consumers reading the spec cannot discover the success contract (status code, response schema). **Recommendation:** add a `200` response with the `Pet` schema.

## DOC-02 — `DELETE /pet/{petId}` documents no success response (Medium)

The spec lists only `400` and `404`; the actual success is `200` with body `{"code":200,"type":"unknown","message":"<petId>"}` (the `message` field merely echoes the id — its shape is also undocumented). **Recommendation:** document the `200` response and its body.

## DOC-03 — Invalid input on `POST /pet` documented as `405` (Low)

`405 Method Not Allowed` is semantically wrong for request-body validation — the conventional code (and the one used by the rest of the spec, e.g. `400 Invalid status value`) is `400 Bad Request`. In practice the API never returns `405` at all: observed responses for invalid input were `200` (BUG-01/03) and `500` (BUG-02), so documentation and behaviour disagree from both directions. **Recommendation:** document `400` and implement it (see BUG-01/02/03).
