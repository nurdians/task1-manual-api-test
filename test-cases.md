# Task 1 — Manual API Test: Test Cases

**Take Home Test – Junior to Medior QA Engineer**

| Field | Value |
|---|---|
| Target application | Swagger Petstore — https://petstore.swagger.io/#/ |
| API base URL | `https://petstore.swagger.io/v2` |
| Specification source | https://petstore.swagger.io/v2/swagger.json (retrieved 12 Nov 2026) |
| Tool | Postman collection `petstore-task1.postman_collection.json`, executed with Postman CLI v1.73.0 |
| Scope | `POST /pet`, `GET /pet/findByStatus`, `DELETE /pet/{petId}` |
| Prepared | 12 Nov 2026 |

## 1. Test Approach

- **Type:** functional API testing (positive + negative), each expected result derived from the official Swagger specification.
- **Shared public environment:** the target is a public demo, so every created pet gets a **timestamped unique name** (e.g. `QA Test Dog 1762991234567`) to keep created records unambiguous.
- **Data flow between cases:**
  - `createdPetId` — stored by TC-POST-01 (collection variable).
  - `deletePetId` — stored by TC-DEL-00 (setup request inside the DELETE folder, so that folder can run standalone).
- **Common assertions:**
  - status code matches the documented/expected result,
  - `Content-Type: application/json` (positive cases that return a body),
  - response time < 3000 ms (public demo on shared infrastructure).
- **Pass / Fail criteria:**
  - **PASS** — actual response matches the expected result.
  - **FAIL** — actual response differs from the specification → candidate for the Bug Report.

## 2. Documented Behaviour Under Test

Extracted from `swagger.json` on 12 Nov 2026:

| Endpoint | Documented responses | Request requirements |
|---|---|---|
| `POST /pet` | `405 – Invalid input` | body required; `Pet` requires `name` + `photoUrls` |
| `GET /pet/findByStatus` | `200 – successful operation`, `400 – Invalid status value` | `status` **required**, type array, enum `available` / `pending` / `sold`, `collectionFormat: multi` |
| `DELETE /pet/{petId}` | `400 – Invalid ID supplied`, `404 – Pet not found` | `petId` **required**, integer int64 |

> ⚠ Notes found while preparing the tests (to be verified against actual responses):
> 1. The specification documents **no success response** for `POST /pet` and `DELETE /pet/{petId}`.
> 2. Invalid input on `POST /pet` is documented as **`405`** (Method Not Allowed) rather than the conventional `400`.
>
> Expected results below note these gaps; anything that does not match the actual response is tracked in `report.md` / `bug-report.md`.

## 3. Test Cases

### 3.1 `POST /pet` — Add a new pet

| ID | Scenario | Payload | Expected result | Priority |
|---|---|---|---|---|
| TC-POST-01 | Create pet with valid data | valid `Pet` (name, category, photoUrls, tags, status) | `200`; JSON body containing new numeric `id` > 0 that echoes the submitted data; `Content-Type: application/json`; < 3000 ms *(success code not documented — DOC gap)* | P1 |
| TC-POST-02 | Create pet missing required `name` | `photoUrls` + `status` only | `405 – Invalid input` (spec: required field missing) | P2 |
| TC-POST-03 | Create pet with invalid `id` type | `"id": "not-a-number"` (string vs int64) | `405 – Invalid input` (spec: invalid input) | P2 |
| TC-POST-04 | Create pet with invalid `status` enum | `"status": "not-a-valid-status"` | `405 – Invalid input` (spec: invalid input) | P2 |

### 3.2 `GET /pet/findByStatus` — Find pets by status

| ID | Scenario | Request | Expected result | Priority |
|---|---|---|---|---|
| TC-GET-01 | Find pets with `status=available` | `?status=available` | `200`; non-empty JSON array; every item has `id`, `name` and `status = "available"`; < 3000 ms | P1 |
| TC-GET-02 | Find pets with `status=pending` | `?status=pending` | `200`; non-empty JSON array; every item `status = "pending"` | P1 |
| TC-GET-03 | Find pets with `status=sold` | `?status=sold` | `200`; non-empty JSON array; every item `status = "sold"` | P1 |
| TC-GET-04 | Missing required `status` parameter | *(no query string)* | `400` (spec: `status` is a required parameter) | P2 |
| TC-GET-05 | Invalid `status` value | `?status=not-a-valid-status` | `400 – Invalid status value` (spec) | P2 |
| TC-GET-06 | Multiple status values | `?status=available&status=sold` | `200`; non-empty array; every item status ∈ {available, sold}; **both** values present (spec: `collectionFormat: multi` → OR semantics) | P2 |

### 3.3 `DELETE /pet/{petId}` — Delete a pet by ID

| ID | Scenario | Request | Expected result | Priority |
|---|---|---|---|---|
| TC-DEL-00 | *Setup* — create the pet to be deleted | `POST /pet` (valid payload) | `200`; stores `deletePetId` for the cases below | P1 |
| TC-DEL-01 | Delete an existing pet | `DELETE /pet/{deletePetId}` | `200` success *(success code not documented — DOC gap)*; < 3000 ms | P1 |
| TC-DEL-02 | Delete the same pet again (verify it is gone) | `DELETE /pet/{deletePetId}` | `404 – Pet not found` (spec) | P1 |
| TC-DEL-03 | Delete a non-existent pet | `DELETE /pet/999999999` | `404 – Pet not found` (spec) | P2 |
| TC-DEL-04 | Delete with invalid ID format | `DELETE /pet/abc` | `400 – Invalid ID supplied` (spec: int64 expected) | P2 |

## 4. Test Data Appendix (request payloads)

All requests use header `Content-Type: application/json` where a body is present.

**TC-POST-01 / TC-DEL-00 — valid pet** (`{{petName}}` / `{{deletePetName}}` are set by the pre-request script with a timestamp suffix):

```json
{
  "name": "{{petName}}",
  "category": { "id": 101, "name": "QA-Dogs" },
  "photoUrls": ["https://example.com/pet/qa-dog.jpg"],
  "tags": [{ "id": 1, "name": "qa-automation" }],
  "status": "available"
}
```

**TC-POST-02 — missing required `name`:**

```json
{
  "photoUrls": ["https://example.com/pet/no-name.jpg"],
  "status": "available"
}
```

**TC-POST-03 — `id` has the wrong type:**

```json
{
  "id": "not-a-number",
  "name": "QA Invalid ID Type",
  "photoUrls": ["https://example.com/pet/invalid-id.jpg"],
  "status": "available"
}
```

**TC-POST-04 — `status` outside the enum:**

```json
{
  "name": "QA Invalid Status",
  "photoUrls": ["https://example.com/pet/invalid-status.jpg"],
  "status": "not-a-valid-status"
}
```

**TC-DEL-00 — pet created for the delete cases:**

```json
{
  "name": "{{deletePetName}}",
  "photoUrls": ["https://example.com/pet/delete-target.jpg"],
  "status": "available"
}
```

**GET / DELETE URLs:**

| ID | URL |
|---|---|
| TC-GET-01 | `{{baseUrl}}/pet/findByStatus?status=available` |
| TC-GET-02 | `{{baseUrl}}/pet/findByStatus?status=pending` |
| TC-GET-03 | `{{baseUrl}}/pet/findByStatus?status=sold` |
| TC-GET-04 | `{{baseUrl}}/pet/findByStatus` |
| TC-GET-05 | `{{baseUrl}}/pet/findByStatus?status=not-a-valid-status` |
| TC-GET-06 | `{{baseUrl}}/pet/findByStatus?status=available&status=sold` |
| TC-DEL-01 / TC-DEL-02 | `{{baseUrl}}/pet/{{deletePetId}}` |
| TC-DEL-03 | `{{baseUrl}}/pet/999999999` |
| TC-DEL-04 | `{{baseUrl}}/pet/abc` |

## 5. How to Run

**Postman GUI**

1. Import `petstore-task1.postman_collection.json` (Import → Files).
2. Open the collection **QA Take-Home Task 1 - Petstore API Test** → **Run** (Collection Runner).
3. Keep the request order as-is (TC-POST-* → TC-GET-* → TC-DEL-*), iterations = 1, then **Run**.

**Postman CLI**

```bash
cd task1-manual-api-test
postman collection run petstore-task1.postman_collection.json --no-report-events --output evidence
```

**Traceability:** every request name in the collection is identical to its TC ID above. Execution results are recorded in `report.md`.
