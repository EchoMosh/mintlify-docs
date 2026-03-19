# ChatGrid Public API — Design Spec

**Date**: 2026-03-19
**Status**: Approved
**Audience**: External developers building on ChatGrid
**Base URL**: `https://api.chatgrid.app/v1`
**Framework**: Hono (TypeScript) on Fly.io

---

## 1. Overview

A versioned REST API that exposes ChatGrid's core functionality to external developers. The frontend continues talking to Supabase directly — this API is a separate surface for third-party integrations.

### What's exposed

| Resource  | Description                                              |
| --------- | -------------------------------------------------------- |
| Boards    | Canvas workspaces containing nodes, edges, and chats     |
| Nodes     | Items on a board (documents, images, embeds, chat nodes) |
| Edges     | Connections between nodes                                |
| Chats     | AI conversation threads, optionally linked to nodes      |
| Messages  | Individual messages within chats (triggers AI responses) |
| Documents | Parse, OCR, vectorize, and search content                |
| Assets    | File uploads (PDFs, images, etc.)                        |
| Jobs      | Async operation status tracking                          |
| User      | Current user profile and credit balance                  |

### What's NOT exposed (v1)

- Viral Vault (future)
- Composio integrations (future)
- Agent triggers (future)
- Community posts / social features
- Billing/Stripe (handled via dashboard)

---

## 2. Authentication

### API Keys

Format: `cgk_live_<32 hex chars>` (production) / `cgk_test_<32 hex chars>` (sandbox)

```
Authorization: Bearer cgk_live_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

- SHA-256 hash stored in `api_keys` table — raw key shown once at creation
- Same header accepts Supabase JWTs (prefix-based dispatch in middleware)
- `cgk_` prefix enables GitHub secret scanning detection

### API Keys Table

```sql
CREATE TABLE api_keys (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  key_hash      TEXT NOT NULL UNIQUE,
  key_prefix    TEXT NOT NULL,
  name          TEXT NOT NULL DEFAULT 'Default',
  scopes        TEXT[] NOT NULL DEFAULT '{read,write}',
  rate_limit    INT NOT NULL DEFAULT 60,
  last_used_at  TIMESTAMPTZ,
  expires_at    TIMESTAMPTZ,
  revoked_at    TIMESTAMPTZ,
  created_at    TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_user ON api_keys(user_id);
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
```

### Scopes

| Scope   | Allows                               |
| ------- | ------------------------------------ |
| `read`  | GET endpoints                        |
| `write` | POST/PATCH/DELETE endpoints          |
| `admin` | API key management, account settings |

Default: `['read', 'write']`. Admin requires explicit opt-in.

---

## 3. Response Format

Every response includes an `object` type discriminator.

### Single resource

```json
{
  "object": "board",
  "data": {
    "id": "board_abc123",
    "name": "Research Project",
    "created_at": "2026-03-19T00:00:00Z",
    "updated_at": "2026-03-19T00:00:00Z"
  }
}
```

### Collection

```json
{
  "object": "list",
  "data": [
    { "id": "board_abc123", "name": "Research Project" },
    { "id": "board_def456", "name": "Meeting Notes" }
  ],
  "has_more": true,
  "cursor": "eyJpZCI6ImJvYXJkX2RlZjQ1NiJ9"
}
```

### Error

```json
{
  "object": "error",
  "status": 404,
  "code": "board_not_found",
  "message": "Board board_abc123 does not exist"
}
```

### Validation error

```json
{
  "object": "error",
  "status": 400,
  "code": "validation_error",
  "message": "Validation failed",
  "details": [{ "path": "name", "message": "Required", "code": "invalid_type" }]
}
```

---

## 4. Pagination

Cursor-based. Default limit: 50. Max limit: 100.

```
GET /v1/boards?limit=20&cursor=eyJpZCI6...&order=desc
```

Parameters:

- `limit` — items per page (1-100, default 50)
- `cursor` — opaque cursor from previous response
- `order` — `asc` or `desc` (by `created_at` unless otherwise noted)

Response always includes `has_more` and `cursor` (null if no more pages).

---

## 5. Rate Limiting

Three layers:

| Layer    | Scope                 | Default        |
| -------- | --------------------- | -------------- |
| Per-key  | `api_keys.rate_limit` | 60 req/min     |
| Per-user | Sum of all keys       | 200 req/min    |
| Global   | Entire API            | 10,000 req/min |

Headers on every response:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1711036800
Retry-After: 23          (only on 429)
```

Credits deducted only on AI calls (message send, document processing), not CRUD operations.

---

## 6. Endpoints

### 6.1 Boards

```
POST   /v1/boards                          Create a board
GET    /v1/boards                          List boards
GET    /v1/boards/:boardId                 Get a board
PATCH  /v1/boards/:boardId                 Update a board
DELETE /v1/boards/:boardId                 Delete a board
POST   /v1/boards/:boardId/duplicate       Duplicate a board
```

**Create board**

```json
POST /v1/boards
{
  "name": "Research Project",
  "view_mode": "canvas"           // "canvas" | "docs"
}
→ 201 { "object": "board", "data": { ... } }
```

**List boards**

```
GET /v1/boards?limit=20&order=desc
→ 200 { "object": "list", "data": [...], "has_more": false }
```

Sortable by: `created_at`, `updated_at`, `name`.

### 6.2 Board Sharing

```
POST   /v1/boards/:boardId/shares          Share a board
GET    /v1/boards/:boardId/shares          List shares
PATCH  /v1/boards/:boardId/shares/:shareId Update share permissions
DELETE /v1/boards/:boardId/shares/:shareId Revoke share
```

**Share a board**

```json
POST /v1/boards/:boardId/shares
{
  "access_level": "viewer"        // "viewer" | "editor"
}
→ 201 { "object": "share", "data": { "id": "...", "share_url": "...", "access_level": "viewer" } }
```

### 6.3 Nodes

```
POST   /v1/boards/:boardId/nodes           Add a node
POST   /v1/boards/:boardId/nodes/batch     Batch create/update/delete
GET    /v1/boards/:boardId/nodes           List nodes
GET    /v1/boards/:boardId/nodes/:nodeId   Get a node
PATCH  /v1/boards/:boardId/nodes/:nodeId   Update a node
DELETE /v1/boards/:boardId/nodes/:nodeId   Delete a node
```

**Create node**

```json
POST /v1/boards/:boardId/nodes
{
  "type": "document",             // "document" | "chat" | "image" | "embed" | "flashcards"
  "position": { "x": 100, "y": 200 },
  "data": {
    "title": "Meeting Notes",
    "content": "..."
  }
}
→ 201 { "object": "node", "data": { ... } }
```

**Batch operations**

```json
POST /v1/boards/:boardId/nodes/batch
{
  "operations": [
    { "action": "create", "data": { "type": "document", "position": { "x": 0, "y": 0 }, "data": {} } },
    { "action": "update", "id": "node_abc", "data": { "position": { "x": 100, "y": 100 } } },
    { "action": "delete", "id": "node_def" }
  ]
}
→ 200 { "object": "batch_result", "data": { "created": 1, "updated": 1, "deleted": 1, "errors": [] } }
```

**List nodes with type filter**

```
GET /v1/boards/:boardId/nodes?type=document,chat&limit=50
```

### 6.4 Edges

```
POST   /v1/boards/:boardId/edges           Connect two nodes
GET    /v1/boards/:boardId/edges           List edges
PATCH  /v1/boards/:boardId/edges/:edgeId   Update an edge
DELETE /v1/boards/:boardId/edges/:edgeId   Remove an edge
```

**Create edge**

```json
POST /v1/boards/:boardId/edges
{
  "source_node_id": "node_abc",
  "target_node_id": "node_def",
  "type": "default",
  "data": {}                      // optional metadata
}
→ 201 { "object": "edge", "data": { ... } }
```

### 6.5 Chats

```
POST   /v1/boards/:boardId/chats                    Create a chat
GET    /v1/boards/:boardId/chats                    List chats
GET    /v1/boards/:boardId/chats/:chatId            Get a chat
PATCH  /v1/boards/:boardId/chats/:chatId            Update (link/unlink node)
DELETE /v1/boards/:boardId/chats/:chatId            Delete a chat
```

**Create chat**

```json
POST /v1/boards/:boardId/chats
{
  "title": "Research Discussion",
  "node_id": "node_abc"          // optional — link to a node
}
→ 201 { "object": "chat", "data": { ... } }
```

**Link/unlink node via PATCH**

```json
PATCH /v1/boards/:boardId/chats/:chatId
{ "node_id": "node_abc" }        // link
{ "node_id": null }              // unlink
```

**Filter by node**

```
GET /v1/boards/:boardId/chats?node_id=node_abc
```

### 6.6 Messages

```
POST   /v1/boards/:boardId/chats/:chatId/messages   Send message
GET    /v1/boards/:boardId/chats/:chatId/messages    List messages
```

**Send message (synchronous)**

```json
POST /v1/boards/:boardId/chats/:chatId/messages
{
  "content": "Summarize the attached documents",
  "stream": false
}
→ 200 {
  "object": "message",
  "data": {
    "id": "msg_abc123",
    "role": "assistant",
    "content": "Here is a summary...",
    "model": "openai/gpt-4o",
    "usage": { "prompt_tokens": 1200, "completion_tokens": 350 },
    "created_at": "2026-03-19T00:00:00Z"
  }
}
```

**Send message (streaming via SSE)**

```json
POST /v1/boards/:boardId/chats/:chatId/messages
{
  "content": "Summarize the attached documents",
  "stream": true
}
→ Content-Type: text/event-stream

event: message.delta
data: {"content": "Here is"}

event: message.delta
data: {"content": " a summary"}

event: message.complete
data: {"id": "msg_abc123", "role": "assistant", "content": "Here is a summary...", "usage": {...}}
```

**List messages**

```
GET /v1/boards/:boardId/chats/:chatId/messages?limit=100&order=asc
```

### 6.7 Documents & Search

```
POST   /v1/boards/:boardId/documents/process     Parse/OCR a document
POST   /v1/boards/:boardId/documents/vectorize    Vectorize content
POST   /v1/boards/:boardId/documents/search       Semantic search
```

**Process document (async — returns job)**

```json
POST /v1/boards/:boardId/documents/process
{
  "file_url": "https://...",       // or upload via assets first
  "node_id": "node_abc"           // optional — attach to node
}
→ 202 {
  "object": "job",
  "data": { "id": "job_abc", "status": "processing", "type": "document_process" }
}
```

**Semantic search**

```json
POST /v1/boards/:boardId/documents/search
{
  "query": "machine learning applications",
  "limit": 10,
  "threshold": 0.7,
  "node_ids": ["node_abc"]        // optional — scope to specific nodes
}
→ 200 {
  "object": "list",
  "data": [
    { "content": "...", "score": 0.92, "node_id": "node_abc", "metadata": {} }
  ]
}
```

### 6.8 Assets

```
POST   /v1/boards/:boardId/assets           Upload a file
GET    /v1/boards/:boardId/assets           List assets
DELETE /v1/boards/:boardId/assets/:assetId  Delete an asset
```

**Upload**

```
POST /v1/boards/:boardId/assets
Content-Type: multipart/form-data
file: <binary>
→ 201 { "object": "asset", "data": { "id": "asset_abc", "url": "...", "size": 1024000, "content_type": "application/pdf" } }
```

### 6.9 Jobs

```
GET    /v1/jobs/:jobId              Check job status
GET    /v1/jobs                     List jobs
```

**Job status lifecycle**: `processing` → `completed` | `failed`

```json
GET /v1/jobs/job_abc
→ 200 {
  "object": "job",
  "data": {
    "id": "job_abc",
    "type": "document_process",
    "status": "completed",
    "result": { "node_id": "node_abc", "chunks_created": 47 },
    "created_at": "...",
    "completed_at": "..."
  }
}
```

### 6.10 User

```
GET    /v1/me                       Get current user
GET    /v1/me/usage                 Get credit balance
```

```json
GET /v1/me
→ 200 {
  "object": "user",
  "data": {
    "id": "user_abc",
    "email": "dev@example.com",
    "created_at": "..."
  }
}

GET /v1/me/usage
→ 200 {
  "object": "usage",
  "data": {
    "credits_remaining": 4200,
    "monthly_allowance": 5000,
    "period_end": "2026-04-01T00:00:00Z"
  }
}
```

---

## 7. Project Structure

```
src/api/
├── index.ts                  # Hono app entry, middleware stack
├── middleware/
│   ├── auth.ts               # API key + JWT dispatch
│   ├── rateLimit.ts          # Sliding window rate limiter
│   └── validate.ts           # Zod validation factory
├── routes/
│   ├── boards.ts
│   ├── nodes.ts
│   ├── edges.ts
│   ├── chats.ts
│   ├── messages.ts
│   ├── documents.ts
│   ├── assets.ts
│   ├── jobs.ts
│   └── me.ts
├── schemas/
│   ├── board.ts              # Zod schemas (feeds OpenAPI gen for Mintlify)
│   ├── node.ts
│   ├── chat.ts
│   ├── message.ts
│   ├── document.ts
│   └── common.ts             # Shared types (pagination, envelope)
└── lib/
    ├── responses.ts          # Envelope helpers (ok, created, error, list)
    ├── supabase.ts           # Service role client
    └── credits.ts            # Credit deduction logic
```

---

## 8. Prerequisites — flow_state Migration

**The `nodes` and `edges` tables exist but have 0 rows.** All node data currently lives as JSON inside `boards.flow_state`. The API cannot safely expose individual node CRUD on top of a JSONB blob (concurrent writes = data corruption).

### Migration plan

1. **Extract script**: Read every board's `flow_state`, insert nodes into `nodes` table, insert edges into `edges` table. Keep `flow_state` intact during transition.
2. **Dual-write**: Frontend saves to both `flow_state` and `nodes`/`edges` tables.
3. **API reads from tables**: The public API only uses the `nodes`/`edges` tables.
4. **Frontend migration**: Move frontend to read from `nodes`/`edges` tables.
5. **Cleanup**: Strip node/edge data from `flow_state`, keep only viewport settings.

This is Phase 1 of the implementation plan.

---

## 9. Security Checklist

- [ ] API keys: SHA-256 hashed, raw shown once, `cgk_` prefix for leak detection
- [ ] Auth middleware: prefix-based dispatch (API key vs JWT), no dev fallbacks
- [ ] All queries scoped by `user_id` (service role key bypasses RLS)
- [ ] Never expose: `key_hash`, other users' emails, internal Composio config
- [ ] CORS: explicit origin allowlist (`chatgrid.app`, `www.chatgrid.app`)
- [ ] Zod validation on every endpoint, structured error responses
- [ ] Rate limit headers on every response
- [ ] Fix existing Composio webhook (no signature verification)
- [ ] Fix existing CORS (wide-open `cors()` with no origin)
- [ ] Outbound webhooks: HMAC-SHA256 signed with timestamp for replay protection

---

## 10. Technology Choices

| Component     | Choice                   | Rationale                                                                             |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------- |
| Framework     | Hono                     | 13KB, TypeScript-first, Zod validation via `@hono/zod-validator`, runs on any runtime |
| Deploy        | Fly.io                   | Already in agreed stack, `api.chatgrid.app` custom domain                             |
| Validation    | Zod                      | Already used throughout codebase, feeds OpenAPI generation                            |
| Auth          | API keys + JWT           | Standard Bearer header, same pattern as OpenAI/Stripe                                 |
| Docs          | Mintlify                 | Already decided — Zod → OpenAPI → Mintlify pipeline                                   |
| Rate limiting | In-memory sliding window | Simple for single Fly.io machine, upgrade to Redis when scaling                       |

---

## 11. Design Decisions Log

| Decision        | Chosen                      | Rejected                       | Rationale                                                              |
| --------------- | --------------------------- | ------------------------------ | ---------------------------------------------------------------------- |
| Framework       | Hono                        | Fastify, Express               | Zod-native, 13KB vs 500KB, multi-runtime, you already use Zod          |
| Naming          | nodes/edges                 | items/connections              | Matches codebase, standard canvas API terminology (Miro uses same)     |
| Chat nesting    | Fully under boards          | Dual-path (boards + top-level) | Chats always belong to a board, consistent URL hierarchy               |
| Link/unlink     | PATCH chat with node_id     | Separate link/unlink endpoints | Standard REST, fewer endpoints                                         |
| Pagination      | Cursor-based                | Offset-based                   | Stable under inserts/deletes, industry standard (OpenAI, Notion, Miro) |
| Response format | `{ object, data }` envelope | Flat responses                 | Enables polymorphic deserialization, matches OpenAI/Notion pattern     |
| Async pattern   | 202 + job polling + SSE     | Webhooks only                  | Polling is simpler for v1, SSE for streaming AI responses              |
| API server      | Separate from server.js     | Bolt onto existing server      | server.js is 4K lines of frontend plumbing, clean separation           |
