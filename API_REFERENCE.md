# RICSA API Integration Reference

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `sqlite:///./data/dev.db` | SQLAlchemy datasource URL. Swap to `postgresql+psycopg://user:pass@host/db` for production |
| `RAGFLOW_BASE_URL` | `https://ragflow.internal.bankraya.co.id` | Base URL of the RAGFlow server |
| `RAGFLOW_API_KEY` | `change-me` | Bearer token for authenticating with the RAGFlow API |
| `RAGFLOW_CHAT_ID` | `change-me` | RAGFlow chat assistant ID used for sessions and completions |
| `RAGFLOW_DATASET_ID` | `change-me` | RAGFlow dataset ID used for document uploads |
| `LOG_LEVEL` | `INFO` | Logging verbosity (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

`.env` example:
```
DATABASE_URL="sqlite:///./data/dev.db"
RAGFLOW_BASE_URL="https://ragflow.internal.bankraya.co.id"
RAGFLOW_API_KEY="ragflow-xxxxxxxxxxxxxxxx"
RAGFLOW_CHAT_ID="a9c38a7c394111f197ce2bd595e42f2e"
RAGFLOW_DATASET_ID="45756a0c393e11f197ce2bd595e42f2e"
LOG_LEVEL="INFO"
```

---

## Constants & Enums

### Role Codes (5 total)

Defined in `app/enums.py`. Every authenticated request carries one role via the `x-personal-number` → `User.role` lookup.

| Code | Name | Description |
|---|---|---|
| `R-IA` | IA User | Read-only: can chat and view their own sessions |
| `R-MK` | Maker | Upload documents, replace files, submit batches |
| `R-CK` | Checker | Review and check submitted batches; can reject |
| `R-SG` | Signer | Approve checked batches (triggers RAGFlow push); can reject |
| `R-AD` | Access Admin | Full access: manage users, view audit logs, delete from RAGFlow, manage access metadata |

### Workflow Status Codes

| Code | Meaning |
|---|---|
| `DRAFT` | Batch created by Maker; files stored locally, not yet in RAGFlow |
| `SUBMITTED` | Maker submitted for review; awaiting Checker |
| `CHECKED` | Checker reviewed; awaiting Signer |
| `APPROVED` | Signer approved; documents queued/pushed to RAGFlow |
| `REJECTED` | Rejected by Checker (from SUBMITTED) or Signer (from CHECKED) |

**Valid state machine transitions:**

```
DRAFT → SUBMITTED    (R-MK, original maker only)
SUBMITTED → CHECKED  (R-CK)
SUBMITTED → REJECTED (R-CK)
CHECKED → APPROVED   (R-SG)  ← triggers background RAGFlow push
CHECKED → REJECTED   (R-SG)
```

### Confidentiality Codes

| Code | Name | Description |
|---|---|---|
| `U` | Umum | Public — accessible to all authenticated users with clearance ≥ 0 |
| `R` | Rahasia | Restricted — requires clearance ≥ 1 AND must be a collaborator or in a matching group |

### Clearance Level

Boolean. Stored on each `User` record.

| Value | Access |
|---|---|
| `false` | Umum (`U`) documents only |
| `true` | Rahasia (`R`) documents, provided the user is also a collaborator or in an access group |

`R-AD` bypasses all clearance checks and sees every document regardless of value.

---

## Database Schema

SQLModel (SQLAlchemy) models defined in `app/models/`. SQLite in dev; PostgreSQL in production.

### user

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `UR-{ULID}` |
| `personal_number` | `VARCHAR(16)` | UNIQUE, indexed | Employee ID; used as the `x-personal-number` header value |
| `name` | `VARCHAR(120)` | NOT NULL | Display name |
| `role` | `ENUM(R-IA, R-MK, R-CK, R-SG, R-AD)` | NOT NULL | Role code |
| `has_clearance` | `BOOLEAN` | DEFAULT FALSE | `true` = can access Rahasia docs (if also collaborator/group member) |
| `is_active` | `BOOL` | DEFAULT TRUE | Soft-delete flag; inactive users are rejected at auth |
| `created_at` | `DATETIME` | DEFAULT NOW | UTC |
| `updated_at` | `DATETIME` | DEFAULT NOW | UTC |

**Relations:** `user_group` (many-to-many → `group`)

### group

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `GRP-{ULID}` |
| `code` | `VARCHAR(32)` | UNIQUE, indexed | e.g. `SKAI`, `EDM` |
| `name` | `VARCHAR(120)` | NOT NULL | Human-readable name |

**Seeded values:** `SKAI` (Satuan Kerja Audit Internal), `EDM` (Desk EDM)

### user_group (junction)

| Column | Type | Constraints |
|---|---|---|
| `user_id` | `VARCHAR` | FK → `user.id`, PK |
| `group_id` | `VARCHAR` | FK → `group.id`, PK |

### document_category (lookup)

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `CAT-{ULID}` |
| `code` | `VARCHAR(16)` | UNIQUE, indexed | e.g. `LHA`, `KKA`, `DPA` |
| `name` | `VARCHAR(120)` | NOT NULL | Full category name |

**Seeded values:** `LHA` (Laporan Hasil Audit), `KKA` (Kertas Kerja Audit), `DPA` (Dokumen Pendukung Audit)

### confidentiality (lookup)

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `CONF-{ULID}` |
| `code` | `ENUM(U, R)` | UNIQUE, indexed | Confidentiality code |
| `name` | `VARCHAR(60)` | NOT NULL | `Umum` or `Rahasia` |
| `clearance_required` | `INT` | DEFAULT 0 | Minimum clearance level required to view |

### document_batch_application_workflow_status (lookup)

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `WS-{ULID}` |
| `code` | `ENUM(DRAFT, SUBMITTED, CHECKED, APPROVED, REJECTED)` | UNIQUE, indexed | Status code |
| `name` | `VARCHAR(60)` | NOT NULL | Display label |

### document_batch

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `DB-{ULID}` |
| `title` | `VARCHAR(240)` | NOT NULL | Batch display title |
| `category_id` | `VARCHAR` | FK → `document_category.id` | |
| `confidentiality_id` | `VARCHAR` | FK → `confidentiality.id` | |
| `workflow_status_id` | `VARCHAR` | FK → `document_batch_application_workflow_status.id`, indexed | Current FSM state |
| `maker_id` | `VARCHAR` | FK → `user.id` | Original uploader |
| `checker_id` | `VARCHAR` | FK → `user.id`, NULL | Set when batch transitions to CHECKED |
| `signer_id` | `VARCHAR` | FK → `user.id`, NULL | Set when batch transitions to APPROVED or REJECTED from CHECKED |
| `rejected_reason` | `VARCHAR(500)` | NULL | Populated on rejection |
| `created_at` | `DATETIME` | DEFAULT NOW | UTC |
| `updated_at` | `DATETIME` | DEFAULT NOW | UTC |

**Relations:** `document` (one-to-many), `document_batch_collaborator` (many-to-many → `user`), `document_group` (many-to-many → `group`)

### document

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `DC-{ULID}` |
| `batch_id` | `VARCHAR` | FK → `document_batch.id` (CASCADE DELETE), indexed | Parent batch |
| `version` | `VARCHAR(8)` | NOT NULL | e.g. `000001` — increments on file replacement |
| `filename` | `VARCHAR(240)` | NOT NULL | Original uploaded filename |
| `storage_path` | `VARCHAR(500)` | NOT NULL | Local disk path: `data/uploads/{batch_id}__{doc_id}__{safe_name}` |
| `mime_type` | `VARCHAR(80)` | NOT NULL | e.g. `application/pdf` |
| `size_bytes` | `INT` | NOT NULL | File size in bytes |
| `ragflow_document_id` | `VARCHAR(80)` | NULL, UNIQUE | Set after the batch is APPROVED and pushed to RAGFlow |
| `ragflow_dataset_id` | `VARCHAR(80)` | NULL | RAGFlow dataset ID (mirrors `RAGFLOW_DATASET_ID` env var) |
| `uploaded_at` | `DATETIME` | DEFAULT NOW | UTC |

### document_batch_collaborator (junction)

| Column | Type | Constraints |
|---|---|---|
| `batch_id` | `VARCHAR` | FK → `document_batch.id` (CASCADE), PK |
| `user_id` | `VARCHAR` | FK → `user.id`, PK |

### document_group (junction)

| Column | Type | Constraints |
|---|---|---|
| `batch_id` | `VARCHAR` | FK → `document_batch.id` (CASCADE), PK |
| `group_id` | `VARCHAR` | FK → `group.id`, PK |

### audit_log

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `VARCHAR` | PK | Format: `LOG-{ULID}` |
| `timestamp` | `DATETIME` | indexed | UTC, auto-set |
| `user_id` | `VARCHAR` | FK → `user.id`, indexed | Actor |
| `endpoint` | `VARCHAR(120)` | | e.g. `/api/ai/chat` |
| `method` | `VARCHAR(10)` | | HTTP verb |
| `session_id` | `VARCHAR(80)` | NULL | RAGFlow session ID (chat calls only) |
| `question` | `TEXT` | NULL | User's question; `[REDACTED]` if any retrieved doc is Rahasia (FR-AI-LOG-02) |
| `retrieved_doc_ids` | `JSON` | NULL | List of local `document.id` values from RAGFlow chunks |
| `access_status` | `VARCHAR(20)` | | `granted` / `partial` / `no_context` / `error` |
| `latency_ms` | `INT` | | End-to-end response time |
| `http_status` | `INT` | | HTTP response status code |
| `error_code` | `VARCHAR(20)` | indexed | Error code if request failed, else null |
| `error_detail` | `TEXT` | NULL | Human-readable detail for the error |

---

## Helper Functions

### Server-side

| Function | File | Description |
|---|---|---|
| `new_id(prefix)` | `app/ids.py` | Generates a prefixed ULID string (e.g. `new_id("DB")` → `DB-01KP...`). Prefixes: `UR` user, `DB` batch, `DC` document, `LOG` audit log, `CAT` category, `CONF` confidentiality, `WS` workflow status, `GRP` group |
| `build_metadata_condition(user)` | `app/services/ragflow_filter.py` | Returns a RAGFlow `metadata_condition` dict for the caller, or `None` for R-AD (sees everything). Encodes clearance + collaborator + group rules as OR conditions |
| `post_filter_chunks(chunks, user)` | `app/services/ragflow_filter.py` | Second-pass filter applied after RAGFlow returns — guards against the RAGFlow bug #12865 metadata leak by re-checking each chunk's `meta_fields` against the caller's clearance/groups |
| `normalize_sources(chunks)` | `app/services/ragflow_filter.py` | Transforms raw RAGFlow chunk dicts into clean `source` objects (document_id, document_name, page, box, similarity, snippet) |
| `create_draft(session, maker, ...)` | `app/services/ingest_service.py` | Creates a `document_batch` + N `document` rows in DRAFT state; saves files to `data/uploads/`; does NOT call RAGFlow |
| `replace_document_file(session, document, ...)` | `app/services/ingest_service.py` | Overwrites the stored file bytes and bumps `version`; if batch is APPROVED also re-pushes to RAGFlow |
| `build_meta_fields(session, batch, document)` | `app/services/ingest_service.py` | Returns the `meta_fields` dict sent to RAGFlow: `{ confidentiality, collaborators, groups, document_id, batch_id }` |
| `push_document(session, batch, document, ragflow)` | `app/services/ingest_service.py` | Uploads one document to RAGFlow, sets metadata, triggers parse, and writes back `ragflow_document_id` |
| `delete_document_from_ragflow(session, document, ragflow)` | `app/services/ingest_service.py` | Calls RAGFlow delete, nulls `ragflow_document_id`. Returns `True` if removed, `False` if the field was already null |
| `submit / check / approve / reject` | `app/services/workflow_service.py` | FSM transition functions; each validates the current state and actor role before writing the new `workflow_status_id` |
| `health_snapshot(session, ragflow)` | `app/services/health_service.py` | Pings RAGFlow and aggregates last-15-min metrics from `audit_log` into a single dict |
| `log_event(session, ...)` | `app/services/audit_service.py` | Inserts one `audit_log` row; called on every `/api/ai/chat` request |

---

## Authentication

All API routes require the `x-personal-number` header. The middleware (`app/middleware/auth.py`) resolves it to a `User` row and rejects inactive users.

| Scenario | Response |
|---|---|
| Header absent | `401 AUTH` — "Missing x-personal-number header" |
| Unknown personal number | `401 AUTH` — "Unknown user" |
| `is_active = false` | `401 AUTH` — "Unknown user" |
| Role not in allowed set | `403 AI-E03` — "Role not permitted for this endpoint" |

---

## Error Codes

All errors return a JSON envelope:

```json
{
  "code": "ERROR_CODE",
  "message": "Human-readable description",
  "detail": { ... }
}
```

| Code         | HTTP | Class                     | Meaning                                                                                                                                            |
| ------------ | ---- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH`       | 401  | `AuthError`               | Missing or unknown `x-personal-number` header                                                                                                      |
| `VALIDATION` | 422  | `ValidationError`         | Request body or path parameter failed validation                                                                                                   |
| `AI-E01`     | 200  | —                         | Warning only (not an error response): retrieval returned no context, or max similarity < 0.3. Returned in the `warning` field of the chat response |
| `AI-E02`     | 409  | `WorkflowError`           | Invalid FSM transition (e.g. trying to submit a batch that is already APPROVED)                                                                    |
| `AI-E03`     | 403  | `AccessError`             | Role or clearance insufficient for the endpoint                                                                                                    |
| `AI-E04`     | 502  | `RagflowUploadError`      | RAGFlow rejected the document upload (OCR / parse failure)                                                                                         |
| `AI-E05`     | 502  | `RagflowUnavailableError` | RAGFlow unreachable, returned 5xx, or timed out                                                                                                    |
| `INTERNAL`   | 500  | `RicsaError`              | Unhandled server exception                                                                                                                         |

---

## Internal APIs

Base URL: `http://localhost:8000` (dev) — set via `{{baseUrl}}` in Postman.

Authentication: every request must include `x-personal-number: <personal_number>` header.

---

### Tag: ai

---

#### `POST /api/ai/ingest`

Upload one or more files as a new document batch in DRAFT state. No RAGFlow interaction at this stage — documents are stored locally until the batch is approved.

**Required role:** `R-MK`

**Content-Type:** `multipart/form-data`

**Form fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `files` | File (repeat per file) | Yes | One or more document files |
| `title` | string (1–240 chars) | Yes | Batch display title |
| `category_code` | string | Yes | Must match a seeded `document_category.code` — e.g. `LHA`, `KKA`, `DPA` |
| `confidentiality_code` | `U` or `R` | Yes | Confidentiality classification |
| `group_codes` | string (repeat per group) | No | Access group codes — e.g. `SKAI`, `EDM` |
| `collaborator_personal_numbers` | string (repeat per user) | No | Personal numbers of collaborators who can access Rahasia documents |

**Response (201 Created):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "document_ids": [
    "DC-01KPZCAB1KBQSA6BX5EE21004Q",
    "DC-01KPZCAB1M7WB7DX7PHTRVEMEJ"
  ],
  "workflow_status": "DRAFT",
  "message": "Upload stored. Submit for review to progress workflow."
}
```

**Error responses:**

| Status | Code         | Condition                                                                     |
| ------ | ------------ | ----------------------------------------------------------------------------- |
| 401    | `AUTH`       | Missing or unknown `x-personal-number`                                        |
| 403    | `AI-E03`     | Caller is not `R-MK`                                                          |
| 422    | `VALIDATION` | No files provided, unknown `category_code`, or unknown `confidentiality_code` |

---

#### `PUT /api/ai/ingest/{batch_id}/documents/{document_id}`

Replace the file bytes of one document in a batch. Only the original batch maker may call this.

If the batch is already `APPROVED`, the old RAGFlow entry is deleted and the new file is immediately re-uploaded and re-parsed. If the batch is not yet APPROVED, only the local file is overwritten.

**Required role:** `R-MK` (and must be the original maker of the batch)

**Content-Type:** `multipart/form-data`

**Form fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | File | Yes | Replacement file |

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the parent batch (e.g. `DB-01KP...`) |
| `document_id` | ID of the document to replace (e.g. `DC-01KP...`) |

**Response (200 OK):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "document_id": "DC-01KPZCAB41743ME01C49JTRRYV",
  "workflow_status": "APPROVED",
  "ragflow_document_id": "rf-abc123"
}
```

`ragflow_document_id` is `null` if the batch was not APPROVED at the time of replacement.

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-MK` or is not the original maker |
| 404 | `VALIDATION` | `batch_id` or `document_id` not found |
| 502 | `AI-E05` | RAGFlow unreachable (only triggered when batch is APPROVED) |

---

#### `PATCH /api/ai/ingest/{batch_id}/meta`

Update the access metadata (confidentiality, groups, collaborators) on any batch. Re-sends the updated `meta_fields` to RAGFlow for every document that already has a `ragflow_document_id`. RAGFlow overwrites — it does not merge.

**Required role:** `R-AD`

**Content-Type:** `application/json`

**Request body (all fields optional — omit to leave unchanged):**
```json
{
  "confidentiality_code": "R",
  "group_codes": ["SKAI", "EDM"],
  "collaborator_personal_numbers": ["100001", "200001"]
}
```

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the target batch |

**Response (200 OK):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "reindexed_count": 3
}
```

`reindexed_count` is the number of documents successfully updated in RAGFlow. `0` if the batch has no APPROVED documents or RAGFlow is unreachable.

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |
| 404 | `VALIDATION` | `batch_id` not found |

---

#### `DELETE /api/ai/ingest/{batch_id}`

Remove every document in the batch from RAGFlow and null each `ragflow_document_id`. Local database records are retained for audit trail.

**Required role:** `R-AD`

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the target batch |

**Response (200 OK):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "ragflow_removed_count": 5
}
```

`ragflow_removed_count` is the number of documents that had a RAGFlow entry and were successfully removed. Already-null entries are skipped and not counted.

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |
| 404 | `VALIDATION` | `batch_id` not found |

---

#### `DELETE /api/ai/ingest/{batch_id}/documents/{document_id}`

Remove a single document from RAGFlow and null its `ragflow_document_id`. The document row is retained.

**Required role:** `R-AD`

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the parent batch |
| `document_id` | ID of the document to remove from RAGFlow |

**Response (200 OK):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "document_id": "DC-01KPZCAB41743ME01C49JTRRYV",
  "ragflow_removed": true
}
```

`ragflow_removed` is `false` when `ragflow_document_id` was already null (nothing to remove).

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |
| 404 | `VALIDATION` | `batch_id` or `document_id` not found |

---

#### `POST /api/ai/chat`

Send a question to the RAGFlow chat assistant. The backend builds an access-control filter from the caller's role, clearance level, and group memberships before forwarding to RAGFlow. The response chunks are post-filtered again as a guard against the RAGFlow bug #12865 metadata leak.

Every call writes one row to `audit_log`. Questions are stored as `[REDACTED]` if any retrieved document carries `confidentiality = R` (per FR-AI-LOG-02).

**Required role:** Any authenticated user

**Content-Type:** `application/json`

**Request body:**

| Field        | Type    | Required | Description                                                                          |
| ------------ | ------- | -------- | ------------------------------------------------------------------------------------ |
| `question`   | string  | Yes      | User's question                                                                      |
| `session_id` | string  | No       | Existing RAGFlow session ID. If null/omitted, a new session is created automatically |
| `stream`     | boolean | No       | `false` only (streaming not yet supported)                                           |
|              |         |          |                                                                                      |

**Response (200 OK):**
```json
{
  "session_id": "sess-abc-123",
  "answer": "Berdasarkan dokumen yang tersedia...",
  "sources": [
    {
      "document_id": "DC-01KP...",
      "document_name": "LHA-2024-Q3.pdf",
      "page": 3,
      "box": [12, 456, 88, 510],
      "similarity": 0.87,
      "snippet": "Temuan material pada kuartal ketiga..."
    }
  ],
  "warning": null
}
```

When retrieval returns no results or max similarity is below `0.3`, the answer is the fixed Indonesian message `"Sumber tidak ditemukan..."` and the `warning` field is populated:
```json
{
  "warning": {
    "code": "AI-E01",
    "message": "Retrieval returned insufficient context"
  }
}
```

**RBAC filter logic applied to `metadata_condition`:**

| Clearance | Role | Sees |
|---|---|---|
| Any | `R-AD` | All documents (no `metadata_condition`) |
| `false` | Any | Umum (`U`) documents only |
| `true` | Any | Umum (`U`) OR (Rahasia (`R`) AND is a collaborator OR is in a matching group) |

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 502 | `AI-E05` | RAGFlow unreachable |

---

#### `GET /api/ai/health`

Returns the current operational status of the service along with last-15-minute aggregate metrics from `audit_log` and a live RAGFlow connectivity ping.

Always returns HTTP `200` even when degraded — so dashboards and uptime monitors do not alert on RAGFlow downtime alone.

**Required role:** Any authenticated user

**Response (200 OK — healthy):**
```json
{
  "status": "ok",
  "ragflow": {
    "reachable": true,
    "latency_ms": 42
  },
  "metrics_last_15min": {
    "requests": 128,
    "errors": 1,
    "error_rate": 0.0078,
    "p50_latency_ms": 210,
    "p95_latency_ms": 980,
    "retrieval_zero_result_count": 3
  }
}
```

**Response (200 OK — degraded, RAGFlow down):**
```json
{
  "status": "degraded",
  "ragflow": {
    "reachable": false,
    "latency_ms": 3305
  },
  "metrics_last_15min": { ... }
}
```

---

#### `GET /api/ai/logs`

Paginated view of the `audit_log` table. Rahasia-redacted questions appear as `[REDACTED]`.

**Required role:** `R-AD`

**Query parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `from` | ISO 8601 datetime | — | Filter logs at or after this timestamp |
| `to` | ISO 8601 datetime | — | Filter logs at or before this timestamp |
| `user_id` | string | — | Filter by `personal_number` or `user.id` |
| `error_code` | string | — | Filter by error code (e.g. `AI-E05`) |
| `page` | int ≥ 1 | `1` | Page number |
| `size` | int 1–500 | `50` | Results per page |

**Response (200 OK):**
```json
{
  "page": 1,
  "size": 50,
  "total": 234,
  "rows": [
    {
      "id": "LOG-01KP...",
      "timestamp": "2026-04-27T10:00:00",
      "user": {
        "id": "UR-000002",
        "personal_number": "100001",
        "name": "Budi Maker"
      },
      "endpoint": "/api/ai/chat",
      "method": "POST",
      "session_id": "sess-abc",
      "question": "Temuan material pada audit Q3 2024?",
      "retrieved_doc_ids": ["DC-01KP..."],
      "access_status": "granted",
      "latency_ms": 234,
      "http_status": 200,
      "error_code": null,
      "error_detail": null
    }
  ]
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |

---

### Tag: workflow

---

#### `POST /api/documents/{batch_id}/submit`

Transition a batch from `DRAFT` → `SUBMITTED`. Only the original maker of the batch may submit.

**Required role:** `R-MK`

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the batch to submit |

**Response (200 OK):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "workflow_status": "SUBMITTED",
  "ragflow_submission": null,
  "rejected_reason": null
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-MK`, or is not the original maker |
| 404 | `VALIDATION` | `batch_id` not found |
| 409 | `AI-E02` | Batch is not in `DRAFT` state |

---

#### `POST /api/documents/{batch_id}/check`

Transition a batch from `SUBMITTED` → `CHECKED`.

**Required role:** `R-CK`

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the batch to check |

**Response (200 OK):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "workflow_status": "CHECKED",
  "ragflow_submission": null,
  "rejected_reason": null
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-CK` |
| 404 | `VALIDATION` | `batch_id` not found |
| 409 | `AI-E02` | Batch is not in `SUBMITTED` state |

---

#### `POST /api/documents/{batch_id}/sign`

Transition a batch from `CHECKED` → `APPROVED` and queue a background RAGFlow upload. Returns `202 Accepted` immediately. Clients should poll `GET /api/documents/{batch_id}` and watch each document's `ragflow_document_id` fill in as the background task completes.

**Required role:** `R-SG`

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the batch to approve |

**Response (202 Accepted):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "workflow_status": "APPROVED",
  "ragflow_submission": "queued",
  "rejected_reason": null
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-SG` |
| 404 | `VALIDATION` | `batch_id` not found |
| 409 | `AI-E02` | Batch is not in `CHECKED` state |

---

#### `POST /api/documents/{batch_id}/reject`

Transition a batch to `REJECTED`. Checker can reject from `SUBMITTED`; Signer can reject from `CHECKED`. Rejected batches are never pushed to RAGFlow.

**Required role:** `R-CK` or `R-SG`

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the batch to reject |

**Content-Type:** `application/json`

**Request body:**
```json
{
  "reason": "Dokumen tidak lengkap, perlu revisi lampiran"
}
```

**Response (200 OK):**
```json
{
  "batch_id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "workflow_status": "REJECTED",
  "ragflow_submission": null,
  "rejected_reason": "Dokumen tidak lengkap, perlu revisi lampiran"
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-CK` or `R-SG` |
| 404 | `VALIDATION` | `batch_id` not found |
| 409 | `AI-E02` | Batch is not in a rejectable state (`SUBMITTED` or `CHECKED`) |

---

#### `GET /api/documents`

Paginated list of document batches, scoped to what the caller can see.

**Visibility rules:**
- `R-AD`: sees all batches
- `R-MK`: sees only their own batches
- `R-CK`, `R-SG`, `R-IA`: sees batches where they are maker, collaborator, or share a group with the batch

**Required role:** Any authenticated user

**Query parameters:**

| Parameter | Type | Description |
|---|---|---|
| `status` | WorkflowStatus enum | Filter by workflow status (e.g. `DRAFT`, `APPROVED`) |
| `maker_id` | string | Filter by maker's user ID (`UR-...`) |
| `category` | string | Filter by category code (e.g. `LHA`) |
| `confidentiality` | `U` or `R` | Filter by confidentiality code |
| `page` | int ≥ 1 | Default `1` |
| `size` | int 1–200 | Default `50` |

**Response (200 OK):**
```json
{
  "page": 1,
  "size": 50,
  "total": 3,
  "rows": [
    {
      "id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
      "title": "pcp-files",
      "category_code": "LHA",
      "confidentiality_code": "U",
      "workflow_status": "CHECKED",
      "maker_id": "UR-000002",
      "checker_id": "UR-000004",
      "signer_id": null,
      "rejected_reason": null,
      "created_at": "2026-04-24T09:14:46.189981",
      "updated_at": "2026-04-24T09:14:46.189999"
    }
  ]
}
```

---

#### `GET /api/documents/{batch_id}`

Full detail for a single batch, including the document list. Same visibility rules as the list endpoint apply.

**Required role:** Any authenticated user

**Path parameters:**

| Parameter | Description |
|---|---|
| `batch_id` | ID of the batch |

**Response (200 OK):**
```json
{
  "id": "DB-01KPZCAB1DDEXE3NHH0GHSXJSB",
  "title": "pcp-files",
  "category_code": "LHA",
  "confidentiality_code": "U",
  "workflow_status": "CHECKED",
  "maker_id": "UR-000002",
  "checker_id": "UR-000004",
  "signer_id": null,
  "rejected_reason": null,
  "created_at": "2026-04-24T09:14:46.189981",
  "updated_at": "2026-04-24T09:14:46.189999",
  "groups": ["SKAI"],
  "collaborators": ["200001"],
  "documents": [
    {
      "id": "DC-01KPZCAB1KBQSA6BX5EE21004Q",
      "version": "000001",
      "filename": "LHA-2024-Q3.pdf",
      "mime_type": "application/pdf",
      "size_bytes": 1024000,
      "ragflow_document_id": null,
      "uploaded_at": "2026-04-24T09:14:46.200000"
    }
  ]
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 404 | `VALIDATION` | `batch_id` not found or not visible to caller |

---

### Tag: sessions

---

#### `GET /api/sessions`

List RAGFlow chat sessions belonging to the caller. Sessions whose last activity (`updated_at`, falling back to `created_at`) is older than 90 days are excluded per BRS §14.2.

**Required role:** Any authenticated user

**Response (200 OK):**
```json
[
  {
    "id": "sess-abc-123",
    "name": "Audit Q3 2024 consultation",
    "created_at": "2026-04-01T09:00:00",
    "updated_at": "2026-04-01T09:15:00"
  }
]
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 502 | `AI-E05` | RAGFlow unreachable |

---

#### `POST /api/sessions`

Create a new RAGFlow chat session scoped to the caller.

**Required role:** Any authenticated user

**Content-Type:** `application/json`

**Request body:**
```json
{
  "name": "Audit Q3 2024"
}
```

`name` is optional. Omit or pass an empty string to use RAGFlow's default.

**Response (201 Created):**
```json
{
  "id": "sess-new-id",
  "name": "Audit Q3 2024",
  "created_at": "2026-04-27T10:00:00",
  "updated_at": "2026-04-27T10:00:00"
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 502 | `AI-E05` | RAGFlow unreachable |

---

#### `DELETE /api/sessions/{session_id}`

Delete a RAGFlow chat session.

**Required role:** Any authenticated user

**Path parameters:**

| Parameter | Description |
|---|---|
| `session_id` | ID of the RAGFlow session to delete |

**Response:** `204 No Content`

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 502 | `AI-E05` | RAGFlow unreachable |

---

### Tag: admin

---

#### `GET /api/users`

List all user records.

**Required role:** `R-AD`

**Response (200 OK):**
```json
[
  {
    "id": "UR-000002",
    "personal_number": "100001",
    "name": "Budi Maker",
    "role": "R-MK",
    "has_clearance": true,
    "is_active": true,
    "group_codes": ["SKAI"],
    "created_at": "2026-04-23T02:48:57.697075",
    "updated_at": "2026-04-23T02:48:57.697081"
  }
]
```

---

#### `POST /api/users`

Create a new user.

**Required role:** `R-AD`

**Content-Type:** `application/json`

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `personal_number` | string | Yes | Must be unique. Max 16 characters |
| `name` | string | Yes | Display name. Max 120 characters |
| `role` | Role code | Yes | One of `R-IA`, `R-MK`, `R-CK`, `R-SG`, `R-AD` |
| `has_clearance` | boolean | No | Default `false` |
| `group_codes` | string[] | No | List of group codes. Unknown codes are ignored |

```json
{
  "personal_number": "100042",
  "name": "Ani",
  "role": "R-MK",
  "has_clearance": true,
  "group_codes": ["SKAI"]
}
```

**Response (201 Created):** Full `UserView` object (same shape as list items above).

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |
| 422 | `VALIDATION` | `personal_number` already exists, or invalid `role` |

---

#### `PATCH /api/users/{user_id}`

Update one or more attributes of a user. All fields are optional; omitted fields are unchanged.

**Required role:** `R-AD`

**Path parameters:**

| Parameter | Description |
|---|---|
| `user_id` | Internal user ID (format: `UR-...`) |

**Content-Type:** `application/json`

**Request body (all optional):**
```json
{
  "name": "Ani Updated",
  "role": "R-CK",
  "has_clearance": true,
  "group_codes": ["SKAI", "EDM"],
  "is_active": false
}
```

**Response (200 OK):** Updated `UserView` object.

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |
| 422 | `VALIDATION` | `user_id` not found |

---

#### `DELETE /api/users/{user_id}`

Soft-delete a user by setting `is_active = false`. The user record is retained. The user immediately loses the ability to authenticate (the auth middleware rejects inactive users).

**Required role:** `R-AD`

**Path parameters:**

| Parameter | Description |
|---|---|
| `user_id` | Internal user ID (format: `UR-...`) |

**Response (200 OK):** `UserView` object with `is_active: false`.

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |
| 422 | `VALIDATION` | `user_id` not found |

---

#### `POST /api/admin/cleanup-sessions`

Delete all RAGFlow sessions whose last activity is older than 90 days. Intended to be called by an infrastructure cron at 23:00 WIB. Per-session errors are silently skipped.

**Required role:** `R-AD`

**Response (200 OK):**
```json
{
  "deleted": 5
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| 401 | `AUTH` | Missing or unknown `x-personal-number` |
| 403 | `AI-E03` | Caller is not `R-AD` |

---

### Tag: lookups

All lookup endpoints require any authenticated user and return a flat list. These are read-only reference data seeded at startup.

---

#### `GET /api/lookups/categories`

**Response (200 OK):**
```json
[
  {"code": "LHA", "name": "Laporan Hasil Audit"},
  {"code": "KKA", "name": "Kertas Kerja Audit"},
  {"code": "DPA", "name": "Dokumen Pendukung Audit"}
]
```

---

#### `GET /api/lookups/confidentiality`

**Response (200 OK):**
```json
[
  {"code": "U", "name": "Umum", "clearance_required": 0},
  {"code": "R", "name": "Rahasia", "clearance_required": 1}
]
```

---

#### `GET /api/lookups/workflow-statuses`

**Response (200 OK):**
```json
[
  {"code": "DRAFT", "name": "Draft"},
  {"code": "SUBMITTED", "name": "Submitted"},
  {"code": "CHECKED", "name": "Checked"},
  {"code": "APPROVED", "name": "Approved"},
  {"code": "REJECTED", "name": "Rejected"}
]
```

---

#### `GET /api/lookups/groups`

**Response (200 OK):**
```json
[
  {"code": "SKAI", "name": "Satuan Kerja Audit Internal"},
  {"code": "EDM", "name": "Desk EDM"}
]
```

---

#### `GET /api/lookups/roles`

**Response (200 OK):**
```json
[
  {"code": "R-IA", "name": "IA User"},
  {"code": "R-MK", "name": "Maker"},
  {"code": "R-CK", "name": "Checker"},
  {"code": "R-SG", "name": "Signer"},
  {"code": "R-AD", "name": "Access Admin"}
]
```

---

## Outgoing APIs (RAGFlow)

All calls use header `Authorization: Bearer {RAGFLOW_API_KEY}`.

Base URL: `{RAGFLOW_BASE_URL}` (e.g. `https://ragflow.internal.bankraya.co.id`)

---

### 1. Upload Document

Upload a file to the configured dataset. Called by `ingest_service.push_document` after a batch is APPROVED.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/documents' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -F 'file=@/path/to/document.pdf'
```

**Response (200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "rf-abc123",
      "name": "document.pdf",
      "size": 102400,
      "type": "pdf"
    }
  ]
}
```

---

### 2. Update Document Metadata

Set `meta_fields` on an uploaded document. Called immediately after upload and also by `PATCH /api/ai/ingest/{batch_id}/meta`.

```bash
curl -X PUT '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/documents/{ragflow_document_id}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "meta_fields": {
      "confidentiality": "R",
      "collaborators": ",100001,200001,",
      "groups": ",SKAI,",
      "document_id": "DC-01KP...",
      "batch_id": "DB-01KP..."
    }
  }'
```

**`meta_fields` schema:**

| Field | Format | Description |
|---|---|---|
| `confidentiality` | `"U"` or `"R"` | Confidentiality code |
| `collaborators` | `",pn1,pn2,"` | Comma-delimited personal numbers with leading/trailing commas. The wrapping commas prevent partial substring collisions when RAGFlow evaluates `contains` conditions |
| `groups` | `",SKAI,EDM,"` | Comma-delimited group codes, same wrapping convention |
| `document_id` | `"DC-01KP..."` | Local document ID for reverse lookup from chat sources |
| `batch_id` | `"DB-01KP..."` | Local batch ID for grouping |

**Response (200):**
```json
{"code": 0}
```

---

### 3. Parse Documents

Trigger OCR/chunking for one or more uploaded documents. Called by `ingest_service.push_document` after metadata is set.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/chunks' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{"document_ids": ["rf-abc123"]}'
```

**Response (200):**
```json
{"code": 0}
```

---

### 4. Delete Document

Remove a document from RAGFlow. Called by `DELETE /api/ai/ingest/...` endpoints.

```bash
curl -X DELETE '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/documents' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{"ids": ["rf-abc123"]}'
```

**Response (200):**
```json
{"code": 0}
```

---

### 5. Create Session

Create a new chat session scoped to a user. Called by `POST /api/sessions`.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/sessions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Audit Q3 2024",
    "user_id": "100001"
  }'
```

`user_id` is set to the caller's `personal_number` (not the internal `UR-...` id).

**Response (200):**
```json
{
  "code": 0,
  "data": {
    "id": "sess-new-id",
    "name": "Audit Q3 2024",
    "user_id": "100001",
    "chat_id": "...",
    "create_time": 1713200000,
    "update_time": 1713200000
  }
}
```

---

### 6. List Sessions

List sessions for a user. Called by `GET /api/sessions`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/sessions?page=1&page_size=100&user_id={personal_number}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Response (200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "sess-abc-123",
      "name": "Audit Q3 2024",
      "user_id": "100001",
      "create_time": 1713200000,
      "update_time": 1713200000
    }
  ]
}
```

---

### 7. Delete Sessions

Delete one or more sessions. Called by `DELETE /api/sessions/{session_id}` and `POST /api/admin/cleanup-sessions`.

```bash
curl -X DELETE '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/sessions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{"ids": ["sess-abc-123"]}'
```

**Response (200):**
```json
{"code": 0}
```

---

### 8. Chat Completion

Send a message to the chat assistant with optional RBAC filtering. Called by `POST /api/ai/chat`.

```bash
# Non-admin user (with metadata_condition)
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/completions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "question": "Temuan material pada audit Q3 2024?",
    "session_id": "sess-abc-123",
    "stream": false,
    "metadata_condition": {
      "logic": "or",
      "conditions": [
        {"name": "confidentiality", "comparison_operator": "is", "value": "U"},
        {"name": "collaborators", "comparison_operator": "contains", "value": ",100001,"},
        {"name": "groups", "comparison_operator": "contains", "value": ",SKAI,"}
      ]
    }
  }'

# R-AD (no metadata_condition — sees everything)
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/completions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "question": "Temuan material pada audit Q3 2024?",
    "session_id": "sess-abc-123",
    "stream": false
  }'
```

**`metadata_condition` structure:**
```json
{
  "logic": "or",
  "conditions": [
    {"name": "<meta_field_name>", "comparison_operator": "<op>", "value": "<val>"}
  ]
}
```

Supported `comparison_operator` values: `is`, `not is`, `contains`, `not contains`, `start with`, `end with`, `empty`, `not empty`, `>`, `<`

**Response (200):**
```json
{
  "code": 0,
  "data": {
    "answer": "Berdasarkan dokumen yang tersedia...",
    "reference": {
      "chunks": [
        {
          "id": "chunk-id",
          "content": "...",
          "document_id": "rf-abc123",
          "document_name": "LHA-2024-Q3.pdf",
          "meta_fields": {
            "confidentiality": "U",
            "collaborators": ",100001,200001,",
            "groups": ",SKAI,",
            "document_id": "DC-01KP...",
            "batch_id": "DB-01KP..."
          },
          "positions": [[3, 12, 456, 88, 510]],
          "similarity": 0.87
        }
      ]
    },
    "session_id": "sess-abc-123"
  }
}
```

> **Known issue:** RAGFlow bug [#12865](https://github.com/infiniflow/ragflow/issues/12865) — `metadata_condition` on chat assistants may leak non-matching documents. RICSA guards against this with `post_filter_chunks()` applied after every completion call.

---

### 9. Fetch Citation Image

Proxy a document chunk image. Note: uses `/v1/` prefix, not `/api/v1/`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/v1/document/image/{imageId}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Response (200):**
- **Content-Type:** `image/png` (or relevant image MIME type)
- **Body:** Raw image binary

**Error behavior:** RAGFlow may return `200 application/json` when the image does not exist. The proxy detects this and returns `404`.
