# BRS vs. Codebase — Progress Checklist

## Context

This codebase is an API integration test harness whose purpose is to prove out the functions and endpoints that will ultimately be handed off to the frontend and backend teams building **RICSA (Raya Intelligent Computer System for Audit System) — AI Chatbot Audit berbasis RAG**.

You asked for a gap analysis against the BRS (`/Users/bankraya/Downloads/BRS AI RICSA.md`) so you can see how far the POC has come and hand a precise punch list to the dev teams. The checklist below walks every clause of the BRS in order and marks whether the current repo satisfies it.

**Source under review:**
- Next.js 16 app @ `/Users/bankraya/Codes/work/ricsa-api-integration-test`
- Prisma + SQLite (`prisma/schema.prisma`)
- 9 API routes under `src/app/api/*`
- 3 pages (`/`, `/admin`, `/documents`)
- RAGFlow as the downstream RAG engine (OCR/layout/chunk/embed/retrieval/LLM all live there)

---

## Legend

| Marker | Meaning |
|---|---|
| ✅ | Done — implemented in this repo |
| ☑️ | Done (delegated) — RAGFlow handles it; repo wires the integration |
| 🟡 | Partial — some aspects implemented, others missing |
| ❌ | Not Done — no implementation in this repo |
| ➖ | Out of scope per BRS §3.2 |

---

## §3.1 — In-Scope Capabilities

| # | Capability | Status | Evidence / Gap |
|---|---|---|---|
| 1 | Document ingestion & validation | 🟡 | Upload works (`/api/datasets/[id]/documents`); no workflow-status gate |
| 2 | Document analysis (OCR, layout, table extraction) | ☑️ | RAGFlow |
| 3 | Chunking & metadata enrichment | ☑️ | RAGFlow; `confidentiality` + `collaborators` meta are set on each doc via `PUT /api/v1/datasets/.../documents/{id}` |
| 4 | Embedding & vector indexing | ☑️ | RAGFlow |
| 5 | Retrieval engine based on user access | ✅ | `metadata_condition` injected in `/api/completions/route.ts:30` |
| 6 | Prompt orchestration & LLM inference | ☑️ | RAGFlow (prompt + model configured there) |
| 7 | Source attribution & explainability | ✅ | References surfaced in `/api/completions` response and rendered in `src/app/page.tsx:312` |
| 8 | Audit logging & telemetry | ❌ | No log table, no log endpoint, no logging middleware |
| 9 | AI service API for Front End | 🟡 | Chat/ingest exist; `/ai/health` + `/ai/logs` missing |

---

## §3.2 — Out of Scope (per BRS)

- ➖ UI/UX design
- ➖ App-level role-menu management
- ➖ SSO / Identity Provider setup

---

## §4 — Backend AI Design Principles

| # | Principle | Status | Notes |
|---|---|---|---|
| 1 | Security by Design | 🟡 | RBAC in `/api/completions`; but `/api/keys` has zero auth, `.env` is gitignored only |
| 2 | Least Privilege Access | 🟡 | Metadata filter enforces per-user doc access; user CRUD is unauthenticated |
| 3 | Explainable & Traceable AI | 🟡 | Citations returned; no audit trail |
| 4 | Human-in-the-loop | ✅ | Auditor initiates queries; no auto-decisioning |
| 5 | No Black Box Decision | 🟡 | Citations ✓; disclaimer/"sumber tidak ditemukan" ✗ |
| 6 | Separation of Concern | ✅ | Clean Prisma / route / RAGFlow split |

---

## §5 — Architectural Components

| # | Component | Status | Where |
|---|---|---|---|
| 1 | API Gateway AI | 🟡 | Next.js route handlers act as gateway; no dedicated gateway layer |
| 2 | Document Ingestion Service | ✅ | `src/app/api/datasets/[datasetId]/documents/route.ts` |
| 3 | Document Analysis Engine | ☑️ | RAGFlow |
| 4 | Embedding Service | ☑️ | RAGFlow |
| 5 | Vector Database | ☑️ | RAGFlow |
| 6 | Retrieval Engine | ☑️ | RAGFlow + `metadata_condition` client-side filter |
| 7 | Prompt & Context Builder | ☑️ | RAGFlow (template lives there — needs Desk EDM to verify) |
| 8 | LLM Inference Service | ☑️ | RAGFlow |
| 9 | Audit Log & Monitoring Service | ❌ | Not implemented |

---

## §7 — Access Control Principles

| # | Rule | Status | Evidence / Gap |
|---|---|---|---|
| 1 | Backend re-verifies user access (not blind-trusting FE) | 🟡 | `userId` resolved to `personalNumber` server-side; but any caller can hit `/api/keys` without auth |
| 2a | Retrieval honors **group akses** | ❌ | No group dimension in schema — only `personal_number` collaborators |
| 2b | Retrieval honors **klasifikasi dokumen** | ✅ | `confidentiality` meta used in filter |
| 2c | Retrieval honors **confidential clearance** | 🟡 | Binary (`superuser` bypass) — not a graduated clearance level |
| 3 | Draft / Submitted / Rejected docs never ingested/embedded/sourced | ❌ | No workflow-state gate at ingest |

---

## §8 — Functional Requirements

### §8.1 Document Ingestion & Validation

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-ING-01 | Accept only docs with status ≥ Checked/Approved | ❌ | `application_workflow_status_id` stored but never validated |
| FR-AI-ING-02 | Store metadata: ID, Title, Category, Status, Group, Clearance | 🟡 | ID/Title/Category/Clearance ✓; Status ∼ stored but unused; **Group akses missing** |
| FR-AI-ING-03 | Split storage/dataset into Biasa / Rahasia | 🟡 | Single dataset + `confidentiality=U/R` metadata; BRS requests a physical split |
| FR-AI-ING-04 | Rahasia docs → security flag + tighter policy + limited logging | 🟡 | Flag via `confidentiality=R` and `collaborators` filter; no extra policy hooks, no log-suppression |
| FR-AI-ING-05 | Each update → new version + reindex trigger | ❌ | `version` field seeded `"000001"`; no update endpoint, no reindex on change |

### §8.2 Document Analysis Engine

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-DA-01 | Detect PDF-text / PDF-scan / Image | ☑️ | RAGFlow |
| FR-AI-DA-02 | OCR + page/position refs (coords optional) | ☑️ | RAGFlow; references include `image_id`, `positions` |
| FR-AI-DA-03 | Layout analysis (title/heading/paragraph/table) | ☑️ | RAGFlow |
| FR-AI-DA-04 | CSV / relational (optional) | ☑️ | RAGFlow |
| FR-AI-DA-05 | Extracted tables as RAG source + semantic query | ☑️ | RAGFlow |
| FR-AI-DA-06 | Analysis results inherit parent doc access metadata | ✅ | `confidentiality` + `collaborators` set on each RAGFlow doc → chunks inherit |

### §8.3 Chunking & Preprocessing

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-CH-01 | Chunking with configurable size | ☑️ | RAGFlow config |
| FR-AI-CH-02 | Per-chunk metadata (docID, page, type, klasifikasi, group, clearance) | ☑️ | Inherited via RAGFlow meta-fields — **group missing** (same gap as FR-AI-ING-02) |

### §8.4 Embedding & Vector Index

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-EMB-01 | Embedding per chunk | ☑️ | RAGFlow |
| FR-AI-EMB-02 | Vector DB separate from main DB | ☑️ | RAGFlow (separate infra) |
| FR-AI-EMB-03 | Auto-reindex on meta change / doc update / doc delete / group change | ❌ | No update/delete routes; no reindex plumbing |

### §8.5 Retrieval Engine

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-RET-01 | Retrieval uses similarity + group + klasifikasi + clearance | 🟡 | Similarity ✓, klasifikasi ✓; **group ✗**, clearance = binary |
| FR-AI-RET-02 | Only returns chunks user is authorized for | ✅ | `metadata_condition` in `/api/completions/route.ts:31` |
| FR-AI-RET-03 | Returned chunks carry citation metadata | ✅ | `document_name`, `image_id`, `similarity` in response |
| FR-AI-RET-04 | Rahasia chunks blocked without clearance | ✅ | Collaborators filter + superuser bypass |

### §8.6 Prompt & Context Builder

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-PR-01 | Grounded answers only (no hallucination beyond docs) | ☑️ | RAGFlow system prompt — **Desk EDM must verify** |
| FR-AI-PR-02 | Prompt includes audit disclaimer | ☑️ | RAGFlow system prompt — **Desk EDM must verify** |
| FR-AI-PR-03 | Sensitive internal metadata not leaked into LLM context | ☑️ | Depends on RAGFlow prompt builder — **Desk EDM must verify** |

### §8.7 LLM Inference

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-LLM-01 | LLM invoked via controlled internal API | ☑️ | RAGFlow |
| FR-AI-LLM-02 | LLM provider configurable (on-prem / private cloud / managed) | ☑️ | RAGFlow model config |

### §8.8 Source Attribution & Explainability

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-SRC-01 | Every answer includes source | ✅ | References rendered in chat UI |
| FR-AI-SRC-02 | Source = doc name + page/section | 🟡 | `document_name` ✓; page/section exists in RAGFlow `positions` but **not surfaced in UI** |

### §8.9 Audit Logging & Monitoring

| Code | Description | Status | Notes |
|---|---|---|---|
| FR-AI-LOG-01 | Log every AI request (userId, timestamp, docs retrieved, access status) | ❌ | Nothing implemented |
| FR-AI-LOG-02 | Logs don't store full-text of Rahasia docs | ❌ | N/A until logging exists |
| FR-AI-LOG-03 | Logs viewable only by Access Admin | ❌ | No log endpoint |
| FR-AI-LOG-04 | Monitoring for error rate / latency / retrieval failure | ❌ | No telemetry |

---

## §9 — API Backend AI Endpoints

| Endpoint | Status | Current mapping |
|---|---|---|
| `POST /ai/chat` | ✅ | `POST /api/completions` |
| `POST /ai/ingest` | ✅ | `POST /api/datasets/[datasetId]/documents` |
| `GET /ai/health` | ❌ | Only `/api/debug` exists (dev-only, exposes config masks) |
| `GET /ai/logs` | ❌ | Not implemented |

Renaming to `/ai/*` paths is a wiring concern — the behavioral surface is two out of four.

---

## §10 — Non-Functional Requirements

| Code | Description | Status | Notes |
|---|---|---|---|
| NFR-AI-SEC-01 | Encryption in-transit & at-rest | 🟡 | HTTPS to RAGFlow ✓; local SQLite **not encrypted**; `.env` gitignored but unencrypted |
| NFR-AI-SEC-02 | Rahasia never sent to public model | ☑️ | RAGFlow model routing — **Desk EDM must verify** |
| NFR-AI-PERF-01 | Retrieval & inference SLA defined | ❌ | No SLA, no monitoring |
| NFR-AI-SCAL-01 | Horizontal scaling supported | 🟡 | Next.js handlers are stateless; SQLite will block multi-instance → Postgres swap needed |
| NFR-AI-GOV-01 | Audit & compliance support | ❌ | No audit logging |

---

## §11 — Error Handling

| Code | Description | Status | Current behavior |
|---|---|---|---|
| AI-E01 | No context matches user access | 🟡 | RAGFlow returns empty `chunks` — not surfaced as explicit error code |
| AI-E02 | Document not eligible | ❌ | No eligibility check exists |
| AI-E03 | Clearance insufficient | 🟡 | Filter silently excludes — no explicit 403 |
| AI-E04 | OCR/table extraction failed | ☑️ | RAGFlow surfaces; no translation to `AI-E04` |
| AI-E05 | LLM service unavailable | 🟡 | 502 `"RAGFlow server unreachable"` — not coded as `AI-E05` |

---

## §12 — Acceptance Criteria

| # | Criterion | Status |
|---|---|---|
| 1 | PDF/Image analyzed & scanned | ☑️ (RAGFlow) |
| 2 | Tables semantically queryable | ☑️ (RAGFlow) |
| 3 | Retrieval only from authorized docs | ✅ |
| 4 | Non-final docs never used as source | ❌ |
| 5 | All AI activity logged | ❌ |
| 6 | Answers always include citation | ✅ |
| 7 | "Sumber tidak ditemukan/tidak memadai" warning when sources insufficient | ❌ |

---

## §14 — Role Menu Matrix

### §14.1 Roles

| Code | Role | Status |
|---|---|---|
| R-IA | IA User | ❌ |
| R-MK | Maker | ❌ |
| R-CK | Checker | ❌ |
| R-SG | Signer | ❌ |
| R-AD | Access Admin | ❌ |

Current schema only has `user` / `superuser`. BRS expects five.

### §14.2 Menu Matrix

| # | Menu | BRS Access | Status | Notes |
|---|---|---|---|---|
| 1 | Beranda RICSA | Semua pengguna | 🟡 | Chat page `/` exists; no branded dashboard |
| 2 | Unggah Dokumen | Maker + Access Admin | 🟡 | `/documents` open to anyone; no role gate |
| 3 | Chat dengan AI | Semua pengguna | ✅ | |
| 4 | Kolom Input Pertanyaan | Semua pengguna | ✅ | |
| 5 | Tampilan Balasan AI | Semua pengguna | ✅ | |
| 6 | Riwayat Interaksi (last 3 months) | Semua pengguna | 🟡 | Sessions list via RAGFlow; **no 3-month cutoff enforced** |
| 7 | Monitoring dan Log AI | Access Admin | ❌ | Not implemented |
| 8 | Pengaturan AI (model/params) | Desk EDM + Access Admin | ❌ | Not implemented |
| 9 | Bantuan & Disclaimer | Semua pengguna | ❌ | Not implemented |

### §14.3 User Matrix / §14.4 Alur Proses

Descriptive — no direct code surface; relevant for RBAC wiring once roles are extended.

---

## Gap Summary by Owner

### 🟦 Backend dev — to build
- Workflow state machine (`Draft → Submitted → Checked → Approved → Rejected`) + submit/review/sign/reject endpoints → unblocks FR-AI-ING-01 / ING-05 / §7.3 / Acceptance #4
- Group-akses dimension on `Document` + metadata + retrieval filter → FR-AI-ING-02 / CH-02 / RET-01
- Audit log model + write middleware for every `/api/completions` and `/api/datasets/*/documents` call → FR-AI-LOG-01/02/03, NFR-AI-GOV-01, Acceptance #5
- `GET /ai/health` (config probe + RAGFlow ping)
- `GET /ai/logs` with Access Admin gate → FR-AI-LOG-03
- Reindex/update/delete endpoints (`PUT/DELETE` on Document) → FR-AI-EMB-03 / ING-05
- Role-system expansion to `R-IA / R-MK / R-CK / R-SG / R-AD` + middleware → §14.1
- 3-month session retention (scheduled cleanup) → §14.2-#6
- Error-code envelope standardization (`AI-E01`..`AI-E05`) → §11
- "Sumber tidak ditemukan/tidak memadai" detection when `chunks.length === 0` or similarity < threshold → Acceptance #7
- Auth on `/api/keys` (currently unauthenticated) → §4.2 (Least Privilege)
- Move from SQLite → Postgres for multi-instance scale → NFR-AI-SCAL-01

### 🟨 Frontend dev — to build
- Role-aware navigation (hide Admin/EDM menus from regular users) → §14.2
- `Bantuan & Disclaimer` page → §14.2-#9
- `Monitoring dan Log AI` view for Access Admin → §14.2-#7
- `Pengaturan AI` view for Desk EDM / Admin → §14.2-#8
- Surface `positions` / page number in citation cards → FR-AI-SRC-02
- Disclaimer banner rendered under every AI answer → FR-AI-PR-02, §14.2-#9
- "Sumber tidak ditemukan" empty-state when backend returns the warning → Acceptance #7
- Role-aware upload form (disable for non-Maker/Admin) → §14.2-#2

### 🟪 RAGFlow / Desk EDM / Infra
- Verify the RAGFlow system prompt implements: (a) grounded answers only, (b) audit disclaimer, (c) no internal metadata leak → FR-AI-PR-01/02/03
- Confirm LLM routing: Rahasia never hits public model endpoints → NFR-AI-SEC-02
- Confirm on-prem / private cloud config per §8.7 → FR-AI-LLM-02
- Define retrieval & inference SLA; expose to `/ai/health` and monitoring → NFR-AI-PERF-01, FR-AI-LOG-04
- Enable encrypted at-rest storage for vector DB + object storage → NFR-AI-SEC-01
- Scheduler @ 23:00 WIB per §14.3 (Divisi INF) — purpose TBD (likely reindex/cleanup)

### ➖ Out of scope per BRS §3.2
- UI/UX design polish
- App-level menu-role management (consumed, not managed, by this backend)
- SSO / IdP setup

---

## Prioritized Next Steps (for this test harness)

If you want the POC to prove out the *last* risky BRS requirements before handing off, attack in this order:

1. **Audit log model + write-path** — biggest BRS gap; unblocks 5 FR codes and Acceptance #5. Add `AuditLog` Prisma model + lightweight wrapper invoked from every `/api/completions` and `/api/datasets/*/documents`.
2. **Workflow state machine** — add `workflow_status` enum + transitions + gate in upload route. Unblocks FR-AI-ING-01, §7.3, Acceptance #4.
3. **Role expansion to 5 BRS roles** — migrate `user/superuser` to `R-IA/R-MK/R-CK/R-SG/R-AD`; adds the vocabulary the frontend will need anyway.
4. **`GET /ai/health` endpoint** — cheap, high-signal for infra handoff.
5. **"Sumber tidak ditemukan" warning** — trivial (check `chunks.length` in `/api/completions`) but closes Acceptance #7.
6. **`GET /ai/logs` endpoint** — depends on #1; gated by the new `R-AD` role from #3.
7. **Group-akses schema + filter** — adds the third RBAC dimension (currently only klasifikasi + collaborator list).
8. **Reindex / update / delete document routes** — FR-AI-EMB-03 + ING-05.
9. **3-month session retention** — cron or lazy filter on session list.
10. **Error-code envelope** — wrap existing 4xx/5xx with `AI-E0N` codes.

Once 1–5 are in place, the POC covers every *behavioral* BRS requirement that the backend team can't defer to RAGFlow.

---

## Verification

This plan is a read-only analysis; nothing in the repo needs to change to verify it. To confirm the status of any line above:

| To verify... | Do this |
|---|---|
| RBAC filter is live | `POST /api/completions` as non-superuser → inspect request RAGFlow receives (has `metadata_condition`) |
| Upload ERD chain writes correctly | `POST /api/datasets/{id}/documents` with a file → `sqlite3 prisma/dev.db ".tables"` shows the 5 tables populated |
| No audit logging exists | `grep -ri "auditlog\|audit_log" src/` → zero hits |
| No workflow-status gating | Upload succeeds regardless of `application_workflow_status_id` value (never validated) |
| No `/ai/health` or `/ai/logs` | `ls src/app/api/` → only the 9 routes listed above |
| Only two roles | `grep -n '"user"\|"superuser"' src/app/api/keys/route.ts:20` |

For RAGFlow-delegated items (☑️), verification lives in the RAGFlow admin UI — not in this repo.
