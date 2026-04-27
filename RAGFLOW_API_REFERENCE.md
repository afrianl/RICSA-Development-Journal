# RAGFlow Outgoing API Reference

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

> **Known issue:** RAGFlow bug [#12865](https://github.com/infiniflow/ragflow/issues/12865) — `metadata_condition` on chat assistants may leak non-matching documents. RICSA should be able to guard against this with a function such as `post_filter_chunks()` which will be applied after every completion call.

**`post_filter_chunks()`:**

```python
def post_filter_chunks(chunks: list[dict], user: RetrievalUser) -> list[dict]:
    if user.role == Role.AD:
        return chunks

    allowed_groups = set(user.group_codes)
    result: list[dict] = []
    for ch in chunks:
        m = ch.get("meta_fields") or {}
        confidentiality = m.get("confidentiality")
        if confidentiality == "U":
            result.append(ch)
            continue
        if not user.has_clearance:
            continue
        if f",{user.personal_number}," in (m.get("collaborators") or ""):
            result.append(ch)
            continue
        groups_meta = m.get("groups") or ""
        if any(f",{g}," in groups_meta for g in allowed_groups):
            result.append(ch)
    return result
```

Each chunk is admitted only if:
- The caller is `R-AD` (all chunks pass through), **or**
- `meta_fields.confidentiality` is `"U"`, **or**
- `meta_fields.confidentiality` is `"R"` **and** `has_clearance` is `true` **and** the caller's `personal_number` appears in `meta_fields.collaborators` **or** any of the caller's group codes appears in `meta_fields.groups`

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
