# RAGFlow Outgoing API Reference

This document is the canonical reference for every RAGFlow HTTP call the RICSA backend makes. Every request and response shown here was captured live against the dev RAGFlow instance (`https://ragflow.dev.internal.rayain.net`). Where the V1 reference and the actual API disagree, this document follows the **actual API** — see "Discrepancies vs. V1" at the bottom for details.

All authenticated calls use header:

```
Authorization: Bearer {RAGFLOW_API_KEY}
```

Base URL: `{RAGFLOW_BASE_URL}` (e.g. `https://ragflow.dev.internal.rayain.net`)

> **HTTP status quirk.** RAGFlow returns **HTTP 200** for almost every error condition (auth failure, missing resource, malformed body) and encodes the real result in a JSON `code` field. Treat `code != 0` as failure regardless of HTTP status. Only routing-level errors (e.g. unknown path) return non-200 HTTP. The error-code reference is at the end of this document.

---

## 1. Health Check

Used by `RagflowClient.ping` (called from `GET /api/ai/health`).

```bash
curl -X GET '{RAGFLOW_BASE_URL}/v1/system/healthz'
```

> Path is `/v1/system/healthz` — **no `/api/` prefix**, and **no auth required**.

**Success (HTTP 200):**
```json
{"db":"ok","doc_engine":"ok","redis":"ok","status":"ok","storage":"ok"}
```

The backend treats anything other than `status == "ok"` as unhealthy and surfaces it as `degraded` in `/api/ai/health`.

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Wrong path (e.g. `/api/v1/system/healthz`) | 404 | `{"code":404,"data":null,"error":"Not Found","message":"Not Found: /api/v1/system/healthz"}` |
| Network unreachable | — | Backend raises `RagflowUnavailableError` (`AI-E05`) |

---

## 2. Upload Document

Upload a single file to the configured dataset. Called by `ingest_service._upload_and_parse` after a batch is APPROVED.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/documents' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -F 'file=@/path/to/document.pdf'
```

**Success (HTTP 200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "c7162eb442b411f1b31643e9187f31a0",
      "name": "lha_q3_2024.pdf",
      "location": "lha_q3_2024.pdf",
      "dataset_id": "fe2e5af834bc11f19a7aa190e07a6320",
      "size": 2327,
      "type": "pdf",
      "suffix": "pdf",
      "chunk_method": "naive",
      "chunk_count": 0,
      "token_count": 0,
      "run": "UNSTART",
      "status": "1",
      "thumbnail": "thumbnail_c7162eb442b411f1b31643e9187f31a0.png",
      "created_by": "09f185f0193211f192ae4316ab633ba3",
      "create_date": "2026-04-28T10:46:03",
      "create_time": 1777347963142,
      "update_date": "2026-04-28T10:46:03",
      "update_time": 1777347963142,
      "parser_config": { "chunk_token_num": 512, "...": "..." }
    }
  ]
}
```

The backend extracts `data[0].id` and treats it as `ragflow_document_id`. Note that `meta_fields` are **not** set by upload — call `PUT /datasets/.../documents/{id}` immediately after to set them (the backend's `upload_document` does this in one logical step).

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Bad / missing API key | 200 | `{"code":109,"data":false,"message":"Authentication error: API key is invalid!"}` |
| Missing `Authorization` header | 200 | `` {"code":0,"data":false,"message":"`Authorization` can't be empty"} `` |
| Malformed `Authorization` header | 200 | `{"code":0,"data":false,"message":"Please check your authorization format."}` |
| Unknown / unowned dataset | 200 | `{"code":100,"data":null,"message":"LookupError(\"Can't find the dataset with ID …!\")"}` |
| No file in form | 200 | `{"code":101,"message":"No file part!"}` |

---

## 3. List Documents

Used by `RagflowClient.find_document_ids_by_batch_id`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/documents?page=1&page_size=100' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Success (HTTP 200):**
```json
{
  "code": 0,
  "data": {
    "total": 4,
    "docs": [
      {
        "id": "c7162eb442b411f1b31643e9187f31a0",
        "name": "lha_q3_2024.pdf",
        "dataset_id": "fe2e5af834bc11f19a7aa190e07a6320",
        "chunk_count": 0,
        "token_count": 0,
        "run": "UNSTART",
        "meta_fields": {},
        "size": 2327,
        "suffix": "pdf",
        "type": "pdf",
        "create_time": 1777347963142,
        "update_time": 1777347963142,
        "parser_config": { "...": "..." }
      }
    ]
  }
}
```

`run` values observed: `UNSTART`, `RUNNING`, `DONE`, `FAIL`. The backend uses `meta_fields.batch_id` to find documents that belong to a given local batch (see `find_document_ids_by_batch_id`).

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Unknown / unowned dataset | 200 | `{"code":102,"message":"You don't own the dataset {id}. "}` |
| Bad / missing API key | 200 | `{"code":109,...}` (see §2) |

---

## 4. Update Document Metadata

Set `meta_fields` on an uploaded document.

```bash
curl -X PUT '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/documents/{ragflow_document_id}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "meta_fields": {
      "confidentiality": "U",
      "collaborators": ",100001,200001,",
      "workspaces": ",SKAI,",
      "document_id": "DC-01KP...",
      "batch_id": "DB-01KP..."
    }
  }'
```

**`meta_fields` schema (the contract `ingest_service.build_meta_fields` writes):**

| Field | Format | Description |
|---|---|---|
| `confidentiality` | `"U"` or `"R"` | Confidentiality code |
| `collaborators` | `",pn1,pn2,"` | Comma-delimited personal numbers, leading and trailing commas. Wrapping commas prevent partial-substring collisions on RAGFlow `contains` matches |
| `workspaces` | `",SKAI,EDM,"` | Comma-delimited workspace codes, same wrapping convention. (Renamed from `groups` — historical chunks may still carry `groups`; see "Discrepancies" below.) |
| `category` | e.g. `"LHA"` | Document category code |
| `document_id` | `"DC-01KP…"` | Local document ID for reverse lookup from chat sources |
| `document_name` | original filename | Original filename for citation display |
| `batch_id` | `"DB-01KP…"` | Local batch ID, used by `find_document_ids_by_batch_id` |
| `version` | `"000001"` | Batch version |
| `workflow_status` | e.g. `"APPROVED"` | Snapshot of batch state at write time |
| `clearance_required` | `"0"` or `"1"` | Mirrors confidentiality row |

**Success (HTTP 200):** RAGFlow echoes the full document object (same shape as §2 success).

```json
{ "code": 0, "data": { "id": "...", "name": "...", "...": "..." } }
```

> ⚠️ **Silent success on missing key.** `PUT` with a body that has **no** `meta_fields` key (e.g. `{"foo":"bar"}`) returns `{"code":0, ...}` — RAGFlow silently makes no change. Don't rely on this for validation; validate client-side.

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Unknown document id | 200 | `{"code":102,"message":"The dataset doesn't own the document."}` |
| Unknown / unowned dataset | 200 | `{"code":102,"message":"You don't own the dataset {id}. "}` |
| Malformed JSON body | 200 | `{"code":100,"data":null,"message":"<BadRequest '400: Bad Request'>"}` |
| Bad API key | 200 | `{"code":109,...}` |

---

## 5. Parse Documents

Trigger OCR/chunking. Called by `ingest_service._upload_and_parse` after meta is set.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/chunks' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{"document_ids": ["c7162eb442b411f1b31643e9187f31a0"]}'
```

The call returns immediately; parsing is asynchronous. Poll list-documents and watch `run` go `UNSTART → RUNNING → DONE`. The backend treats parse-trigger failures as warnings (see `ingest_service._upload_and_parse`) and does not roll back the upload.

**Success (HTTP 200):**
```json
{"code": 0}
```

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| `document_ids` missing or empty | 200 | `` {"code":102,"message":"`document_ids` is required"} `` |
| One or more unknown document ids | 200 | `{"code":102,"message":"Documents not found: ['…']"}` |
| Bad API key | 200 | `{"code":109,...}` |

---

## 6. Delete Document(s)

Remove documents from RAGFlow. Called by `ingest_service.delete_document_from_ragflow`.

```bash
curl -X DELETE '{RAGFLOW_BASE_URL}/api/v1/datasets/{RAGFLOW_DATASET_ID}/documents' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{"ids": ["c7162eb442b411f1b31643e9187f31a0"]}'
```

**Success (HTTP 200):**
```json
{"code": 0}
```

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Already-deleted / unknown id | 200 | `{"code":102,"message":"Documents not found: ['…']"}` |
| `ids: []` (empty list) | 200 | `{"code":0}` — no-op, no error |
| `{}` (no `ids` key) | 200 | `{"code":0}` — **see warning below** |
| Bad API key | 200 | `{"code":109,...}` |

> ⚠️ **DESTRUCTIVE — empty-body behavior.** Sending `{}` (or any body without an `ids` key) returns `{"code":0}` and is interpreted by RAGFlow as a no-op for documents in our testing — but the same shape on `DELETE …/sessions` (§9) **deletes every session on the chat assistant**. Do not rely on either being safe — always send a non-empty `ids` list explicitly. The backend should validate `len(ids) > 0` before issuing a DELETE.

The backend's idempotent retry-safe deletion path is correct: it treats `code: 102 "Documents not found"` as success on retry (see `delete_document` in `ragflow_client.py`). Other non-zero codes raise `RagflowUploadError`.

---

## 7. Create Session

Open a chat session. Called by `chat_service.chat` when no `session_id` is supplied.

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/sessions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{"name": "Audit Q3 2024"}'
```

`name` is optional — when omitted, RAGFlow auto-generates one (e.g. `"New session"`). The backend currently sends `{}`; it does not pass `user_id` (V1 doc was wrong about this — `user_id` comes back as `""` on the response).

**Success (HTTP 200):**
```json
{
  "code": 0,
  "data": {
    "id": "97fddad242b411f1b31643e9187f31a0",
    "chat_id": "a9c38a7c394111f197ce2bd595e42f2e",
    "name": "Audit Q3 2024",
    "user_id": "",
    "messages": [
      {"role": "assistant", "content": "Hi! I'm your assistant. What can I do for you?"}
    ],
    "create_date": "2026-04-28T10:44:43",
    "create_time": 1777347883673,
    "update_date": "2026-04-28T10:44:43",
    "update_time": 1777347883673
  }
}
```

The backend keeps `data.id` as `session_id`.

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Unknown / unowned chat assistant | 200 | `{"code":102,"message":"You do not own the assistant."}` |
| Malformed JSON body | 200 | `{"code":100,"data":null,"message":"<BadRequest '400: Bad Request'>"}` |
| Bad API key | 200 | `{"code":109,...}` |

---

## 8. List Sessions

Used by `GET /api/sessions` and the cleanup cron at `POST /api/admin/cleanup-sessions`.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/sessions?page=1&page_size=100' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Success (HTTP 200):**
```json
{
  "code": 0,
  "data": [
    {
      "id": "6b61f10042b111f1b31643e9187f31a0",
      "chat_id": "a9c38a7c394111f197ce2bd595e42f2e",
      "name": "SESSION-20260428102159",
      "user_id": "001726",
      "messages": [
        {"role": "assistant", "content": "Hi! I'm your assistant. What can I do for you?"}
      ],
      "create_date": "2026-04-28T10:22:00",
      "create_time": 1777346520342,
      "update_date": "2026-04-28T10:22:00",
      "update_time": 1777346520342
    }
  ]
}
```

When there are no sessions: `{"code":0,"data":[]}`.

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Unknown / unowned chat assistant | 200 | `{"code":102,"message":"You don't own the assistant {id}."}` |
| Bad API key | 200 | `{"code":109,...}` |

---

## 9. Delete Session(s)

Called by `DELETE /api/sessions/{session_id}` and `POST /api/admin/cleanup-sessions`.

```bash
curl -X DELETE '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/sessions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{"ids": ["97fddad242b411f1b31643e9187f31a0"]}'
```

> ⚠️ **Deletion is body-based only.** `DELETE /api/v1/chats/{chat_id}/sessions/{session_id}` (path-based) returns `{"code":100,"data":null,"message":"<MethodNotAllowed '405: Method Not Allowed'>"}`. RAGFlow only supports the body-based form with an `ids` array.

**Success (HTTP 200):**
```json
{"code": 0}
```

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Path-based DELETE (single id in URL) | 200 | `{"code":100,"data":null,"message":"<MethodNotAllowed '405: Method Not Allowed'>"}` |
| Already-deleted / unknown id | 200 | `{"code":102,"message":"The chat doesn't own the session {id}"}` |
| `ids: []` | 200 | `{"code":0}` — no-op |
| `{}` (no `ids` key) | 200 | `{"code":0}` — **deletes every session on the chat** |
| Bad API key | 200 | `{"code":109,...}` |

> ⚠️ **CRITICAL.** Sending `DELETE …/sessions` with body `{}` (no `ids` key) returns `{"code":0}` and **wipes every session** belonging to the chat assistant. Verified live: a list-sessions before showed 3 sessions; sending `{}` left `data: []`. Always send an explicit non-empty `ids` array. Backend should reject empty/missing `ids` client-side before issuing the call.

---

## 10. Chat Completion

Called by `chat_service.chat` (which is invoked from `POST /api/ai/chat`). Uses RAGFlow's native completion endpoint — the OpenAI-compatible variant at `/api/v1/chats_openai/{id}/chat/completions` is **not** used because it doesn't return `reference.chunks` (which the backend needs for citations and the `post_filter_chunks` RBAC guard).

```bash
curl -X POST '{RAGFLOW_BASE_URL}/api/v1/chats/{RAGFLOW_CHAT_ID}/completions' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}' \
  -H 'Content-Type: application/json' \
  -d '{
    "question": "Apa temuan LHA Q3 2024?",
    "stream": false,
    "session_id": "edbcb18c42b411f1b31643e9187f31a0",
    "metadata_condition": {
      "logic": "or",
      "conditions": [
        {"name":"confidentiality","comparison_operator":"is","value":"U"},
        {"name":"collaborators","comparison_operator":"contains","value":",100001,"},
        {"name":"workspaces","comparison_operator":"contains","value":",SKAI,"}
      ]
    }
  }'
```

`metadata_condition` is omitted when the caller is `R-AD` — admins see everything. `session_id` is optional but recommended (preserves multi-turn context).

**Success (HTTP 200):**
```json
{
  "code": 0,
  "data": {
    "id": "5dc930d9-2271-4ae5-969f-ee889eab6ab6",
    "session_id": "edbcb18c42b411f1b31643e9187f31a0",
    "answer": "Berdasarkan dokumen yang tersedia…",
    "audio_binary": null,
    "created_at": 1777348050.5728264,
    "prompt": "You are an intelligent assistant…",
    "reference": {
      "chunks": [
        {
          "id": "chunk-id",
          "content": "...",
          "document_id": "rf-abc123",
          "document_name": "LHA-2024-Q3.pdf",
          "image_id": "rf-abc123-img-0",
          "meta_fields": {
            "confidentiality": "U",
            "collaborators": ",100001,200001,",
            "workspaces": ",SKAI,",
            "document_id": "DC-01KP...",
            "batch_id": "DB-01KP..."
          },
          "positions": [[3, 12, 456, 88, 510]],
          "similarity": 0.87
        }
      ]
    }
  }
}
```

`RagflowClient.completion` unwraps the response and returns `data` directly, so callers see `answer`, `reference`, `session_id`, etc. at the top level. `code != 0` is raised as `RagflowUnavailableError`.

When no documents match (low similarity, no retrieval, or filtered out): `reference.chunks` is `[]` and `answer` becomes the localized "not found" string from the assistant prompt (live: `"The answer you are looking for is not found in the knowledge base!"`).

The `prompt` field is the full assembled prompt RAGFlow sent to the LLM, including timing/token-usage breakdown — useful for debugging retrieval but should not be exposed to end users.

**Supported `comparison_operator` values:** `is`, `not is`, `contains`, `not contains`, `start with`, `end with`, `empty`, `not empty`, `>`, `<`.

> **Known issue: RAGFlow #12865.** `metadata_condition` is unreliable — non-matching chunks may still leak through. The backend always re-applies `post_filter_chunks(chunks, user)` on the result before returning sources to the client (see `ragflow_filter.py`).

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Unknown / unowned chat assistant | 200 | `{"code":102,"message":"You don't own the chat {id}"}` |
| Missing `question` | 200 | `{"code":101,...,"message":"required argument are missing: question; "}` |
| Bad API key | 200 | `{"code":109,...}` |
| Stream/timeout — RAGFlow unreachable | — | Backend raises `RagflowUnavailableError` (`AI-E05`) |

---

## 11. Fetch Citation Image

Proxy a document chunk image. Called by `RagflowClient.fetch_image`. Use the `image_id` from `reference.chunks[].image_id` returned by §10.

> **Path is `/v1/document/image/{imageId}`, NOT `/api/v1/...`** — the `/api/` prefix returns 404 from the routing layer.

```bash
curl -X GET '{RAGFLOW_BASE_URL}/v1/document/image/{imageId}' \
  -H 'Authorization: Bearer {RAGFLOW_API_KEY}'
```

**Success (HTTP 200):**
- `Content-Type`: `image/png` (or whatever MIME the chunk image is)
- Body: raw image bytes

**Image-not-found is the trickiest error in this API.** RAGFlow returns **`HTTP 200 application/json`** with a `code: 102` body even though the request semantically failed:

```
HTTP/2 200
content-type: application/json
content-length: 42

{"code":102,"message":"Image not found."}
```

The backend's proxy must (and does, in `RagflowClient.fetch_image`) sniff the response: if the body starts with `{`, parse it as JSON and treat it as 404 (`RagflowImageNotFound`). Don't trust `Content-Type` alone — RAGFlow does not set `application/json` reliably enough to dispatch on.

**Errors:**

| Case | HTTP | Body |
|---|---|---|
| Image id does not exist | 200 | `application/json` `{"code":102,"message":"Image not found."}` |
| Wrong path (e.g. `/api/v1/document/image/...`) | 404 | `{"code":404,"data":null,"error":"Not Found","message":"Not Found: …"}` |
| Bad API key | 200 | `{"code":109,...}` |


Alternatively, you can just load the image by using the URL as the src property's value in the DOM. One example would be:
```javascript
<Image src='{RAGFLOW_BASE_URL}/v1/document/image/{imageId}'/>
```

---

## Error Code Reference

RAGFlow's body-level `code` field:

| `code` | Meaning                                                                                              | Triggers we observed                                                                        |
| ------ | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `0`    | Success — **also returned for some no-op / silent-failure cases.** Always check the rest of the body | normal success; empty-id deletes; missing-`Authorization` header (with `data: false`)       |
| `100`  | Generic bad request — usually a Werkzeug exception wrapped as a string                               | malformed JSON; `405 MethodNotAllowed` (path-based session delete); unknown dataset id      |
| `101`  | Required argument missing                                                                            | upload with no `file` part; completion without `messages`                                   |
| `102`  | Resource not found / not owned                                                                       | unknown chat / dataset / document / session / image; missing `document_ids`; RBAC ownership |
| `109`  | Authentication failure                                                                               | invalid API key                                                                             |
| `404`  | Routing-level not found (HTTP 404, not 200)                                                          | typo in path                                                                                |
