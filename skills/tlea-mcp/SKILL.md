---
name: tlea-mcp
description: Use the Technology Library East Asia (TLEA) MCP server — 9 read-only tools (search, resolve, get_entity, get_entities, list_records, facets, stats, list_gaps, list_terms) over a curated East Asian energy-technology evidence base with zh/ja/ko termbase resolution. Covers the session handshake (initialize → mcp-session-id), authentication (Access JWT or estate key with catalog:read), the per-kind filter contract, CJK query strategy, and the edge gotchas (Cloudflare 1010 on python-urllib, 421 host allow-list, mandatory trailing slash). Use when asked to query TLEA, tlea.hilir.app, tlea-api.hilir.app or the hilir-tlea plugin, or to find East Asian energy technology records, standards, vendors, projects or documents.
---

# TLEA MCP (Technology Library East Asia)

TLEA is a **read-only** evidence base of East Asian energy technology records — sources,
documents, technologies, patents, vendors, equipment, projects and standards. The MCP door
exposes the same repository the REST API serves (`/api/v1/*` is the primary surface); there
are **no write tools** and the contract test fails if one ever appears. Never attempt a
mutation through it.

## Connection

| | |
|---|---|
| Production MCP | `https://tlea-api.hilir.app/mcp` (the edge normalizes to `/mcp/`; **use the trailing slash everywhere**) |
| Production REST | `https://tlea-api.hilir.app/api/v1/*` |
| Local MCP | `http://127.0.0.1:8000/mcp/` (also :8002; **other ports answer `421 Invalid Host header`** — FastMCP `allowed_hosts`) |
| Auth header | `Authorization: Bearer <credential>` — a hilir Access JWT with `aud` = a registered resource origin (`https://tlea.hilir.app` / `https://tlea-api.hilir.app`), **or** a person's estate key with the `catalog:read` scope |
| Accept header | `application/json, text/event-stream` (mandatory — responses arrive as SSE `data: ` lines) |

Every tool requires `catalog:read`. A valid credential without it → `403 {"error":"forbidden","required":"catalog:read"}`; missing/expired/wrong-audience → `401 {"error":"unauthorized"}`.

Credential rules that were deliberately settled and must not be "simplified":

- **There are no `sk-` keys.** They were deleted (D36); never reintroduce them.
- **Any valid, unexpired estate key with `catalog:read` passes the MCP door.** There is no
  per-door key registration — a 401 from an old key means the key is expired/revoked, not
  "unregistered".
- `x-api-key` is the internal edge→container secret only. **A public request sending it is
  rejected.**
- The `hilir-tlea` ZCode plugin channel authenticates itself with its own credential
  (`hilirToken` in `~/.zcode/cli/plugin-secrets.json`); if the plugin 401s, its key has
  lapsed — the fix is minting a new estate key in the Access account page, not extracting
  the shown-once secret.

## Session flow (raw client recipe)

Streamable HTTP, session-based: initialize once, then carry `mcp-session-id` on every call.

```bash
BASE="https://tlea-api.hilir.app/mcp"
TOKEN="…"   # Access JWT or estate key with catalog:read
H=(-H "Authorization: Bearer $TOKEN" -H "accept: application/json, text/event-stream" \
   -H "content-type: application/json")

# 1. initialize — capture the session id from the response headers
SID=$(curl -fsS -D - -o /dev/null "${H[@]}" -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"agent","version":"1.0"}}}' "$BASE/" | tr -d '\r' | sed -n 's/^[Mm][Cc][Pp]-[Ss][Ee][Ss][Ss][Ii][Oo][Nn]-[Ii][Dd]: //p')
# 2. notifications/initialized (no id, no response expected)
curl -fsS "${H[@]}" -H "mcp-session-id: $SID" -d '{"jsonrpc":"2.0","method":"notifications/initialized"}' "$BASE/"
# 3. tools/list
curl -fsS "${H[@]}" -H "mcp-session-id: $SID" -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' "$BASE/"
# 4. tools/call
curl -fsS "${H[@]}" -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"facets","arguments":{}}}' "$BASE/"
```

If an MCP client (not raw curl) shows an old tool list, that is a **per-session tool
cache** — reconnect and it refreshes. A short description is a stale cache, not an old server.

## The 9 tools

| Tool | Purpose |
|---|---|
| `search(q, kind?, sector?, discipline?, region?, reuse_level?, lang?, limit?)` | Hybrid keyword+vector search. CJK queries are resolved through the termbase first; an unresolved CJK phrase is still searched by meaning via embeddings. Returns `rows`, `total`, and the `resolution` that was applied |
| `resolve(q, lang?)` | The diagnostic door: how a query would resolve (termbase / alias / semantic) and the English query `search` would actually run. Call it when a CJK query underperforms — the fix is termbase vocabulary, not rephrasing |
| `get_entity(kind, id)` | One record by kind + id; ids are stable slugs from `search`/`list_records` |
| `get_entities(kind, ids)` | Batch fetch, ≤20 slugs; found/missing split, duplicates collapsed, unknown kind raises |
| `list_records(kind, sector?, country?, status?, q?, filters?, limit?, offset?)` | Browse a kind without ids. Returns `count` (page), `total` (all matches), `offset`, and is **filter-honest**: `applied` vs `ignored` say which filters reached the query |
| `facets()` | Valid filter codes for every kind. **Call first** — a wrong code returns zero rows silently, never an error |
| `stats()` | Counts by kind, translation coverage, last load — the cheapest check a layer exists at all |
| `list_gaps(kind?, field?, q?, limit?)` | The content-QA gap queue: which record is missing what, why, and where to look next. The acquisition pass reads this before planning a batch |
| `list_terms(lang?, q?, limit?)` | The zh-Hans/zh-Hant/ja/ko termbase the resolver uses. Check exact vocabulary here **before composing a CJK query** |

Eight kinds: `source`, `document`, `technology`, `patent`, `vendor`, `equipment`, `project`,
`standard`. An unknown kind raises with the valid list — never return "nothing found" for a
misspelled kind.

## The agent loop

```
facets()                      → learn the valid sector/discipline/region codes
list_records(kind, sector=…)  → browse what exists (offset/total for paging)
get_entity(kind, id)          → or get_entities for a batch of up to 20
— or, when the id is unknown —
list_terms(lang=…, q=…) → search(q) → get_entity(kind, id from the rows)
```

`search` is not a free-text-only tool: name a `kind` or `sector` when you know one, or the
vector pool (capped at 50 per kind) will rank across everything. Read `rank` for order —
rows are fused keyword+vector, and `score` is a per-engine diagnostic that is **not**
comparable across engines. Use `matched_by` for provenance (`name`/`alias`/`body`/`vector`/`hybrid`).

## Reading the labels

Every row carries licence and quality metadata; quote it with any figure you take:

- `licence` / `reuse_level` / `permitted_use` — what may be reused. Pass `reuse_level`
  explicitly when a consumer must only see reusable text (`full-text-open` vs
  `full-text-restricted` vs `summary-only` vs `catalog-only`).
- `restricted: true` is a **label, not a filter** — filtered content still appears with the
  label. Never strip or downplay it in a summary.
- `verification`, `quality_tier`, `last_verified` — provenance strength.
- `claim: true` marks a vendor-published figure; say so when citing it.
- Documents marked `full-text-restricted` return metadata only — that is a licence
  boundary, not a fetch failure. Don't retry it.

Per-kind `list_records` filters (common four named; the rest inside `filters`):

| Kind | Filters |
|---|---|
| `source` | `code`, `region`, `reuse_level`, `source_class`, `verification`, `sector` |
| `document` | `source_id`, `language`, `reuse_level`, `translation_status` |
| `technology` | `category`, `sector` |
| `patent` | `jurisdiction`, `applicant`, `status`, `family_id`, `sector` |
| `vendor` | `role`, `sector`, `country` |
| `equipment` | `category`, `sector` |
| `project` | `country`, `status`, `sector` |
| `standard` | `issuer`, `status`, `sector` |

`licence_class` (`open`/`restricted`/`unknown`) is the normalised axis on sources and
documents. A filter a kind cannot use comes back in `ignored`, not silently dropped —
`{"kind":"document","sector":"WWT"}` → `applied: [], ignored: ["sector"]`. Don't
re-issue it; browse with a different kind or filter.

## Edge gotchas (each has cost a debugging session)

- **Cloudflare error 1010 blocks python-urllib fingerprints** on `tlea-api.hilir.app` (curl
  passes). Any scripted HTTP probe uses curl or a normal browser User-Agent, never bare
  urllib.
- **`/mcp` without the trailing slash** is 307-normalized at the edge (works), but locally
  FastMCP requires `/mcp/` outright.
- **`421 Invalid Host header`** means the Host is not in FastMCP's allow-list (locally: a
  port other than :8000/:8002; in the runtime-test harness on :8014, rewrite the Host to
  `127.0.0.1:8000`).
- Don't propose a second introspection cache on the client side — verdicts are already
  cached 60 s server-side, and instant revocation is a deliberate guarantee, not latency.
- 400 = unknown kind / bad parameter; 503 = database or internal-key problem (an outage,
  never "revoked").

## When the MCP layer is unavailable

The REST surface answers the same questions with the same credential and needs no session:

```bash
curl -fsS -H "Authorization: Bearer $TOKEN" "https://tlea-api.hilir.app/api/v1/search?q=%E5%8A%A0%E6%B0%A2%E8%A3%82%E5%8C%96&limit=5"
curl -fsS -H "Authorization: Bearer $TOKEN" "https://tlea-api.hilir.app/api/v1/technologies/<id>"
```

Endpoints mirror the tools: `/stats`, `/facets`, `/gaps`, `/terms`, `/resolve`,
`/search`, `/sources|documents|technologies|patents|vendors|equipment|projects|standards`
(list + `/{id}`). `q` ≤ 500 chars, `limit` 1–100, lists carry `X-Total-Count`. Errors use
`{"error":"..."}` with no stack trace, no SQL, no query text — the same is true of MCP
responses; don't echo query text anywhere it could be logged as an error message.
