# RICSA API Integration Test — Reference

## Environment Variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | Prisma datasource URL. Default: `file:./dev.db` (SQLite) |
| `RAGFLOW_BASE_URL` | Base URL of the RAGFlow server |
| `RAGFLOW_API_KEY` | Bearer token for authenticating with the RAGFlow API |
| `RAGFLOW_CHAT_ID` | RAGFlow chat assistant ID used for sessions and completions |
| `RAGFLOW_DATASET_ID` | Default RAGFlow dataset ID; served to the upload page via `GET /api/datasets` |
`.env` example:
```
DATABASE_URL="file:./dev.db" RAGFLOW_BASE_URL="https://ragflow.dev.internal.rayain.net" RAGFLOW_API_KEY="ragflow-hUf_W0uGPKOs4DIDKF9sMIKC1Gpi8Y6mohAokw84oa0" RAGFLOW_CHAT_ID="a9c38a7c394111f197ce2bd595e42f2e" RAGFLOW_DATASET_ID="45756a0c393e11f197ce2bd595e42f2e"
```

---

## Constants

| Constant       | File                                                                                                                 | Source                                 | Description                                                              |
| -------------- | -------------------------------------------------------------------------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------ |
| `CHAT_ID`      | `src/lib/assistants.ts`                                                                                              | `process.env.`<br>`RAGFLOW_CHAT_ID`    | Single RAGFlow chat assistant ID used by sessions and completions routes |
| `DATASET_ID`   | `src/lib/assistants.ts`                                                                                              | `process.env.`<br>`RAGFLOW_DATASET_ID` | Default dataset ID; exposed to the frontend via `GET /api/datasets`      |
| `RAGFLOW_BASE` | `src/app/api/`<br>`{completions,sessions,datasets/`<br>`[datasetId]/documents,image/`<br>`[imageId],debug}/route.ts` | `process.env.`<br>`RAGFLOW_BASE_URL`   | RAGFlow server base URL (module-level const per route file)              |
| `RAGFLOW_KEY`  | Same files as `RAGFLOW_BASE`                                                                                         | `process.env.`<br>`RAGFLOW_API_KEY`    | RAGFlow API bearer token (module-level const per route file)             |


---

## Database Schema (Udah disesuaikan sama ERD)

Aligned with the RICSA ERD. External FK references (to HRD, ISDM, Jagat Raya databases) are stored as plain strings.

### User (local auth simulation — not in ERD)

| Column | Type | Default | Description |
|---|---|---|---|
| `id` | `String` (cuid) | auto | Primary key |
| `name` | `String` | — | Display name |
| `personal_number` | `String` (unique) | — | Maps to `personal_number` in `document_batch_collaborator`; references employee table in external ISDM database |
| `role` | `String` | `"user"` | `"user"` or `"superuser"`. Superusers bypass metadata filters |
| `created_at` | `DateTime` | `now()` | Creation timestamp |

### document_batch

| Column                                 | Type              | Nullable | Description                                           |
| -------------------------------------- | ----------------- | -------- | ----------------------------------------------------- |
| `id`                                   | `varchar(100)` PK | Not Null | Format: `DCMB-[YYYYMM]-[increment]-[version]`         |
| `application_workflow`<br>`_status_id` | `varchar(40)`     | Not Null | References workflow status in <br>Jagat Raya database |
| `workspace_organization_id`            | `varchar(10)`     | Not Null | References organization in HRD database               |
| `workspace_branch_id`                  | `varchar(10)`     | Not Null | References branch in HRD <br>database                 |
| `group_id`                             | `varchar(40)`     | Not Null | Format: `DCMB-[YYYYMM]-[increment]`                   |
| `version`                              | `varchar(6)`      | Not Null | Increments on revision                                |
| `title`                                | `text`            | Not Null | Batch title (e.g. "Hasil Audit CB Menara Brilian")    |
| `note`                                 | `text`            | Null     | Rejection notes etc.                                  |
| `action_at`                            | `timestampz`      | Not Null | Insert/update timestamp                               |
| `action_by`                            | `varchar(40)`     | Not Null | Actor (personal_number/System<br>/Migration)          |

**Relations:** `documents` (Document[]), `workflowStatuses` (DocumentBatchApplicationWorkflowStatus[])

### document

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `varchar(100)` PK | Not Null | Format: `[document_batch_id]-[application_media_id]` |
| `document_batch_id` | `varchar(40)` FK | Not Null | References `document_batch.id` |
| `document_category_id` | `varchar(40)` FK | Not Null | References `document_category.id` |
| `confidentialiy_id` | `varchar(40)` FK | Not Null | References `confidentiality.id` (typo preserved from ERD) |
| `application_media_id` | `varchar(40)` | Not Null | References `application_media` in Jagat Raya database |
| `action_at` | `timestampz` | Not Null | Insert/update timestamp |
| `action_by` | `varchar(40)` | Not Null | Actor |
| `ragflow_document_id` | `String` (unique) | Null | RAGFlow document ID (integration field, not in ERD) |
| `ragflow_dataset_id` | `String` | Null | RAGFlow dataset ID (integration field, not in ERD) |

**Relations:** `documentBatch` (DocumentBatch), `documentCategory` (DocumentCategory), `confidentiality` (Confidentiality)

### confidentiality

| Column | Type | Nullable | Description                                         |
| ------------- | ----------------- | -------- | --------------------------------------------------- |
| `id` | `varchar(100)` PK | Not Null | Format: `CFDY[increment]`                           |
| `name` | `varchar(255)` | Not Null | Short code: `U` (Umum/Public), `R` (Rahasia/Secret) |
| `description` | `text` | Not Null | Human-readable description                          |
| `action_at` | `timestampz` | Not Null | Insert/update timestamp                             |
| `action_by` | `varchar(40)` | Not Null | Actor                                               |

**Seeded values:** `CFDY000001` (U — Umum), `CFDY000002` (R — Rahasia)

### document_category

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `varchar(100)` PK | Not Null | Format: `DCM[sequence]` |
| `name` | `varchar(255)` | Not Null | Short code: `LHA`, `KKA`, `DPA` |
| `description` | `text` | Not Null | Full name (e.g. "Laporan Hasil Audit") |
| `action_at` | `timestampz` | Not Null | Insert/update timestamp |
| `action_by` | `varchar(40)` | Not Null | Actor |

**Seeded values:** `DCM000001` (LHA), `DCM000002` (KKA), `DCM000003` (DPA)

### document_batch_application_workflow_status

| Column                                 | Type              | Nullable | Description                                                    |
| -------------------------------------- | ----------------- | -------- | -------------------------------------------------------------- |
| `id`                                   | `varchar(100)` PK | Not Null | Format: `[document_batch_id]-[application_workflow_status_id]` |
| `document_batch_id`                    | `varchar(100)` FK | Not Null | References `document_batch.id`                                 |
| `application_workflow`<br>`_status_id` | `varchar(40)`     | Not Null | References workflow status in Jagat Raya database              |
| `action_at`                            | `timestampz`      | Not Null | Insert/update timestamp                                        |
| `action_by`                            | `varchar(40)`     | Not Null | Actor                                                          |

**Relations:** `documentBatch` (DocumentBatch), `collaborators` (DocumentBatchCollaborator[])

### document_batch_collaborator

| Column                                                | Type              | Nullable | Description                                                      |
| ----------------------------------------------------- | ----------------- | -------- | ---------------------------------------------------------------- |
| `id`                                                  | `varchar(100)` PK | Not Null | Format: `[workflow_status_id]-[personal_number]`                 |
| `document_batch_application`<br>`_workflow_status_id` | `varchar(100)` FK | Not Null | References `document_batch_application`<br>`_workflow_status.id` |
| `personal_number`                                     | `varchar(10)`     | Not Null | References employee in ISDM database                             |
| `action_at`                                           | `timestampz`      | Not Null | Insert/update timestamp                                          |
| `action_by`                                           | `varchar(40)`     | Not Null | Actor                                                            |

**Relationship chain:** `document_batch_collaborator` -> `document_batch_application_workflow_status` -> `document_batch` <- `document`

---

## Helper Functions

### Server-side

| Function                                  | File                                                        | Description                                                                                                                       |
| ----------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `generateIds(fileCount, collaboratorPNs)` | `src/app/api/datasets/`<br>`[datasetId]/documents/route.ts` | Generates structured IDs for the full ERD chain (batch, N documents, workflow status, collaborators) using ERD naming conventions |
| `pad(n, len)`                             | Same file                                                   | Zero-pads a number to a given length (buat pn biar bisa convert dari, e.g., 1878 jadi 001878)                                     |
| `mask(s)`                                 | `src/app/api/debug/route.ts`                                | Masks a string for safe display: first 12 + last 4 chars with total length (buat masking API Key for debug purposes)              |

### Client-side

| Function                   | File (from my demo project, can be ignored) | Description                                                                     |
| -------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------- |
| `parseReferences(raw)`     | `src/app/page.tsx:47`                       | Transforms raw RAGFlow reference chunks into clean `Reference[]` objects        |
| `fetchUsers()`             | `src/app/page.tsx:69`                       | Fetches user list from `GET /api/keys`, auto-selects first user                 |
| `fetchSessions()`          | `src/app/page.tsx:78`                       | Fetches sessions from `GET /api/sessions?userId=...`                            |
| `createSession()`          | `src/app/page.tsx:129`                      | Creates session via `POST /api/sessions`, sets as active                        |
| `deleteSession(sessionId)` | `src/app/page.tsx:147`                      | Deletes session via `DELETE /api/sessions`                                      |
| `sendMessage(e)`           | `src/app/page.tsx:160`                      | Sends message via `POST /api/completions`, parses response + references         |
| `fetchUsers()`             | `src/app/admin/page.tsx:18`                 | Fetches user list for admin table                                               |
| `createUser(e)`            | `src/app/admin/page.tsx:30`                 | Creates user with name + personalNumber + role                                  |
| `deleteUser(id)`           | `src/app/admin/page.tsx:44`                 | Deletes user by id                                                              |
| `toggleCollaborator(pn)`   | `src/app/documents/page.tsx:72`             | Toggles a personal number in the collaborator set                               |
| `upload(e)`                | `src/app/documents/page.tsx:79`             | Builds FormData with ERD fields and uploads batch of documents (multiple files) |


---

## Internal APIs (Next.js Route Handlers)

### Users

#### `GET /api/keys`

List all users.

- **Response:** `UserRecord[]` — each has `id`, `name`, `personalNumber`, `role`, `createdAt`

#### `POST /api/keys`

Create a user.

- **Body:** `{ "name": string, "personalNumber": string, "role"?: "user" | "superuser" }`
- **Validation:** name + personalNumber required (400), role must be valid (400), personalNumber must be unique (409)
- **Response (201):** `UserRecord`

#### `DELETE /api/keys`

Delete a user.

- **Body:** `{ "id": string }`
- **Response:** `{ "ok": true }`

---

### Lookup Data

#### `GET /api/confidentiality`

List all confidentiality levels.

- **Response:** `Confidentiality[]` — each has `id`, `name`, `description`, `actionAt`, `actionBy`

#### `GET /api/categories`

List all document categories.

- **Response:** `DocumentCategory[]` — each has `id`, `name`, `description`, `actionAt`, `actionBy`

#### `GET /api/datasets`

Returns the configured default dataset ID.

- **Response:** `{ "datasetId": string }`

---

### Documents

#### `POST /api/datasets/:datasetId/documents`

Upload one or more documents to RAGFlow as a batch and create the full ERD chain in the local database.

- **Content-Type:** `multipart/form-data`
- **Path params:** `datasetId` — RAGFlow dataset ID
- **Form fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | File (multiple) | Yes | One or more document files (repeat the `file` field per file) |
| `personalNumber` | string | Yes | Uploader's personal number |
| `confidentialityId` | string | Yes | FK to `confidentiality.id` |
| `documentCategoryId` | string | Yes | FK to `document_category.id` |
| `title` | string | Yes | Batch title |
| `collaboratorPNs` | string | No | JSON array of personal numbers, e.g. `'["001726","001727"]'` |

- **Flow:**
  1. Upload each file to RAGFlow `POST /api/v1/datasets/{datasetId}/documents` (sequentially)
  2. Set metadata on each: `PUT /api/v1/datasets/{datasetId}/documents/{docId}` with `meta_fields: { confidentiality: "<name>", collaborators: ",pn1,pn2," }`
  3. Create one `document_batch` row
  4. Create one `document` row per file (all linked to the same batch, category, confidentiality)
  5. Create one `document_batch_application_workflow_status` row
  6. Create `document_batch_collaborator` rows (uploader always auto-included)

- **RBAC behavior:**
  - Collaborators are **always** stored (uploader auto-included) regardless of confidentiality level, for audit trail purposes
  - Confidentiality `U` (Umum): document accessible to all via `metadata_condition` — collaborators recorded but not restrictive
  - Confidentiality `R`: only collaborators + superusers can access via `metadata_condition` filter

- **Response (201):**
  ```json
  {
    "batch": {...},
    "documents": [...],
    "workflowStatus": {...},
    "collaborators": ["001726", "001727"],
    "ragflowDocuments": [{ "id": "...", "name": "...", "fileIndex": 0 }]
  }
  ```
  `collaborators` is a flat list of personal number strings.

---

### Sessions

#### `GET /api/sessions?userId=:userId`

List RAGFlow chat sessions for a user.

- **Query:** `userId` (required) — internal user ID (cuid). The backend resolves `user.personalNumber` and passes it as RAGFlow's `user_id`.
- **Response:** RAGFlow response proxied: `{ "code": 0, "data": Session[] }`

#### `POST /api/sessions`

Create a new RAGFlow chat session.

- **Body:** `{ "userId": string, "name"?: string }`
- **Note:** RAGFlow's `user_id` is set to the user's `personalNumber` (not the internal cuid), keeping it consistent with the collaborator metadata.
- **Response:** `{ "code": 0, "data": { "id": "...", ... } }`

#### `DELETE /api/sessions`

Delete sessions.

- **Body:** `{ "userId": string, "sessionIds": string[] }`
- **Response:** RAGFlow response proxied

---

### Completions

#### `POST /api/completions`

Send a message to the RAGFlow chat assistant with RBAC filtering.

- **Body:** `{ "userId": string, "sessionId": string, "content": string }`

- **RBAC logic:**
  - **Superuser:** no `metadata_condition` — sees all documents
  - **Regular user:** injects `metadata_condition`:
    ```json
    {
      "logic": "or",
      "conditions": [
        { "name": "confidentiality", "comparison_operator": "is", "value": "U" },
        { "name": "collaborators", "comparison_operator": "contains", "value": ",{personalNumber}," }
      ]
    }
    ```

- **Response:** `{ "code": 0, "data": { "answer": "...", "reference": { "chunks": [...] }, "session_id": "..." } }`

---

### Images

#### `GET /api/image/:imageId`

Proxy a RAGFlow citation image.

- **Response:** Image binary with `Content-Type` + `Cache-Control: public, max-age=3600`, or 404

---

### Debug

#### `GET /api/debug`

Diagnostic endpoint — shows config (API key masked) and tests RAGFlow connectivity.

- **Tests:** owned chats, chat access, dataset access

---

## Outgoing APIs (RAGFlow)

All calls use header `Authorization: Bearer {RAGFLOW_API_KEY}`.

Base URL: `{RAGFLOW_BASE_URL}` (e.g. `https://ragflow.dev.internal.example.net`)

---

### 1. Upload Document

Upload a file to a dataset. Called by `POST /api/datasets/:datasetId/documents`.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/datasets/{datasetId}/documents' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -F 'file=@/path/to/document.pdf'
```

**Request:**
- **Method:** `POST`
- **Content-Type:** `multipart/form-data`
- **Body:** `file` — the file to upload

**Response (200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "abc123def456",
      "name": "document.pdf",
      "size": 102400,
      "type": "pdf",
      "created_by": "...",
      "create_time": 1713200000,
      "update_time": 1713200000
    }
  ]
}
```

---

### 2. Update Document Metadata

Set `meta_fields` on an uploaded document. Called after upload by `POST /api/datasets/:datasetId/documents`.

```bash
curl -X PUT '{RAGFLOW_BASE_URL}/api/v1/datasets/{datasetId}/documents/{documentId}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "meta_fields": {
      "confidentiality": "R",
      "collaborators": ",001726,001727,"
    }
  }'
```

**Request:**
- **Method:** `PUT`
- **Content-Type:** `application/json`
- **Body:**

| Field | Type | Description |
|---|---|---|
| `meta_fields.confidentiality` | `string` | `"U"` (Umum/Public) or `"R"` (Rahasia) |
| `meta_fields.collaborators` | `string` | Comma-delimited personal numbers with leading/trailing commas: `",pn1,pn2,"`. The wrapping commas prevent partial substring collisions when using `contains` matching. |

**Response (200):**
```json
{
  "code": 0
}
```

---

### 3. List Sessions

List chat sessions for a user. Called by `GET /api/sessions` and `GET /api/debug`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/api/v1/chats/{chatId}/sessions?page=1&page_size=100&user_id={personalNumber}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Request:**
- **Method:** `GET`
- **Query params:**

| Param | Type | Description |
|---|---|---|
| `page` | `int` | Page number (1-based) |
| `page_size` | `int` | Results per page |
| `user_id` | `string` | User's `personalNumber` — scopes sessions to this user |

**Response (200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "session-id-here",
      "name": "New Session",
      "messages": [...],
      "chat_id": "...",
      "user_id": "001726",
      "create_time": 1713200000,
      "update_time": 1713200000
    }
  ]
}
```

---

### 4. Create Session

Create a new chat session. Called by `POST /api/sessions`.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{chatId}/sessions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "New Session",
    "user_id": "001726"
  }'
```

**Request:**
- **Method:** `POST`
- **Content-Type:** `application/json`
- **Body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | No | Session display name (defaults to `"New Session"`) |
| `user_id` | `string` | Yes | User's `personalNumber` — ties this session to a specific user |

**Response (200):**
```json
{
  "code": 0,
  "data": {
    "id": "new-session-id",
    "name": "New Session",
    "messages": [
      { "role": "assistant", "content": "Hi! ..." }
    ],
    "chat_id": "...",
    "user_id": "001726",
    "create_time": 1713200000,
    "update_time": 1713200000
  }
}
```

---

### 5. Delete Sessions

Delete one or more sessions by ID. Called by `DELETE /api/sessions`.

```bash
curl -X DELETE '{RAGFLOW_BASE_URL}/api/v1/chats/{chatId}/sessions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "ids": ["session-id-1", "session-id-2"]
  }'
```

**Request:**
- **Method:** `DELETE`
- **Content-Type:** `application/json`
- **Body:**

| Field | Type | Description |
|---|---|---|
| `ids` | `string[]` | Array of session IDs to delete |

**Response (200):**
```json
{
  "code": 0
}
```

---

### 6. Chat Completion

Send a message to the chat assistant with optional RBAC filtering. Called by `POST /api/completions`.

```bash
# Regular user (with RBAC metadata_condition)
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{chatId}/completions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "question": "Apa hasil audit cabang Menara Brilian?",
    "session_id": "existing-session-id",
    "stream": false,
    "metadata_condition": {
      "logic": "or",
      "conditions": [
        { "name": "confidentiality", "comparison_operator": "is", "value": "U" },
        { "name": "collaborators", "comparison_operator": "contains", "value": ",001726," }
      ]
    }
  }'

# Superuser (no metadata_condition — sees all documents)
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{chatId}/completions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "question": "Apa hasil audit cabang Menara Brilian?",
    "session_id": "existing-session-id",
    "stream": false
  }'
```

**Request:**
- **Method:** `POST`
- **Content-Type:** `application/json`
- **Body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `question` | `string` | Yes | The user's message |
| `session_id` | `string` | Yes | Session ID for stateful conversation |
| `stream` | `boolean` | No | `false` for single JSON response (default in this app) |
| `metadata_condition` | `object` | No | RBAC filter (omitted for superusers) |

**`metadata_condition` structure:**
```json
{
  "logic": "or",
  "conditions": [
    { "name": "<meta_field_name>", "comparison_operator": "<op>", "value": "<val>" }
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
          "document_id": "...",
          "document_name": "LHA_Menara_Brilian.pdf",
          "image_id": "optional-image-id",
          "positions": [...]
        }
      ]
    },
    "session_id": "existing-session-id"
  }
}
```

> **Known issue:** RAGFlow bug [#12865](https://github.com/infiniflow/ragflow/issues/12865) — `metadata_condition` on chat assistants may leak non-matching documents. Still open as of v0.24.0. If smoke testing reveals leaked docs, fallback plan: use `/api/v1/retrieval` + separate LLM call.

---

### 7. Fetch Citation Image

Proxy a document citation image. Called by `GET /api/image/:imageId`.

> **Note:** This endpoint uses the `/v1/` prefix, NOT `/api/v1/`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/v1/document/image/{imageId}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Request:**
- **Method:** `GET`
- **No body.**

**Response (200):**
- **Content-Type:** `image/png` (or other image MIME type)
- **Body:** Raw image binary

**Error behavior:** RAGFlow may return `200` with `Content-Type: application/json` when the image doesn't exist. The proxy checks for this and returns `404`.

---

### 8. List Owned Chat Assistants (Debug)

List all chat assistants owned by the API key. Called by `GET /api/debug`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/api/v1/chats?page=1&page_size=100' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Request:**
- **Method:** `GET`
- **Query params:** `page`, `page_size`

**Response (200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "a9c38a7c394111f197ce2bd595e42f2e",
      "name": "RICSA Assistant",
      "description": "...",
      "create_time": 1713200000,
      "update_time": 1713200000
    }
  ]
}
```

---

### 9. List Dataset Documents (Debug)

List documents in a dataset. Called by `GET /api/debug`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/api/v1/datasets/{datasetId}/documents?page=1&page_size=1' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Request:**
- **Method:** `GET`
- **Query params:** `page`, `page_size`

**Response (200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "doc-id",
      "name": "document.pdf",
      "size": 102400,
      "type": "pdf",
      "meta_fields": {
        "confidentiality": "R",
        "collaborators": ",001726,001727,"
      }
    }
  ]
}
```
