---
name: resources-mcp
description: Use the 5M Resources MCP server (resources.hilir.app) to resource-load an EPC schedule with traceable numbers. 39 tools over 32,656 man-hour norms, 52,947 Money-class cost rates, 2,372 escalation indices, 3,155 raw source facts, labour/material prices and benchmarks. It answers "man-hours, crew, plant, duration, cost" with sourced figures plus a seal, builds P6/MSP schedule XML from norms, escalates rates with a named index, writes a Basis of Estimate, and records actuals for calibration. Covers the stateless handshake, Access auth and per-tool scopes, exact-match filter vocabulary, source basis and citation rules, restricted-licence publication gate, the product rules (never invent/convert/escalate silently), and routing to atur/mygis/techlib for work outside its domain. Use for 5M Resources, resources.hilir.app, or to find norms, cost rates, escalation factors, raw tender facts, consumption quantities, or build estimates/schedules/BoE for construction and process-industry projects.
---

# 5M Resources MCP (resource norms)

5M Resources is a **deterministic** norm catalog — Man, Machine, Material, Minutes, Money
records for oil & gas, petrochemical, energy and new-energy projects — exposed as remote MCP
at `POST https://resources.hilir.app/mcp`, REST `/api/v1/*`, and a web UI. The MCP server
teaches the agent: its `initialize` response carries an `instructions` string built from the
catalog actually loaded, so an agent that reads only that still works correctly.

**The cardinal rule: never invent a number.** If the catalog has no value for a question, the
answer is "the catalog holds no record for this" (some empty tools return an explicit
acquisition plan) — never an approximation, interpolation or converted rate. Unverifiable
values belong in the repo's `data/sources/GAPS.md`, not in an estimate.

## What this server gives you — and how you benefit

- **A defensible answer to "how many man-hours, what crew, what plant, how long?"** — `estimate_resources`
  returns low/base/high MH, crew by trade, equipment, duration and a seal; `verify_seal` re-proves it later.
- **A resource-loaded WBS and a schedule file** — `list_wbs_templates` / `suggest_wbs` → `get_logic_template`
  → `build_schedule_block` → `export_schedule` (`p6_xml` / `msp_xml`, with a stated working calendar).
- **Honest cost figures** — `get_cost_norms` keeps place, currency and price year; `list_escalation_indices`
  → `escalate_cost` moves a figure with a NAMED index, never by default; `get_cost_baseline` restates rates;
  `get_benchmarks` gives top-down sanity checks.
- **A reviewable Basis of Estimate** — `export_basis_of_estimate` puts quantities, norms, factors,
  assumptions, seals and the restricted-publication flag into one Markdown document.
- **A closed learning loop** — `log_actuals` records real outcomes; `get_calibration` / `list_actuals`
  show actual-vs-norm ratios so a norm can be flagged, not silently rewritten.
- **Traceability for every number** — provenance, status and confidence on every norm, plus **3,155 raw
  source facts** (`class:'facts'`: 3,128 verbatim RATOL tender observations with item/project/date and 27
  sourced consumption quantities) for historical inference and comparison.

You benefit because figures are sourced, sealed and replayable: an estimate or report can be audited
instead of guessed. Exact-key vocabularies stop an empty page being mistaken for missing data, and
when the catalog does not hold a value it says so and records the gap.

## Not this server's job — route it, do not improvise

- **CPM, float, Monte Carlo P50/P80, earned value, delay/EOT, look-ahead, state holidays and approval lead
  times → atur** (`atur.hilir.app`). The calendars here are only the hours/day and working-week basis used
  on export; atur owns the planning calendar.
- **Site facts** (zoning, flood, slope, access, haul distance) → **mygis**; **contract clauses**
  (PAM/FIDIC/JKR 203A) → **Hilir Kontrak**; **engineering standards, vendors, equipment** → **techlib**;
  **market outlook** → **Market Intel**.
- Auth/scopes never change here — `catalog:read`, `estimate:run`, `actuals:write`, `norms:contribute`
  and admin-only `norms:review` are owned by hilir Access.

## Connection

| | |
|---|---|
| Production MCP | `POST https://resources.hilir.app/mcp` — Streamable HTTP, **stateless** (no `mcp-session-id`; a one-shot `tools/call` works with no session) |
| Discovery | `GET https://resources.hilir.app/.well-known/oauth-protected-resource` (RFC 9728) |
| Local MCP | stdio via `npm run mcp` in the repo (same tools; actuals bound to a local JSONL store) |
| Auth header | `Authorization: Bearer <credential>` **or** `x-api-key: <key>` — a hilir Access JWT (`aud = https://resources.hilir.app`) **or** a person's estate key, introspected at Access (RFC 7662) |
| REST fallback | `https://resources.hilir.app/api/v1/*` — same credential, no session |

One credential family for both MCP and REST — never a separate key type. **There are no
`sk-` keys** (they were deliberately deleted; never reintroduce or invent one).

Scopes are the boundary, and they are enforced **inside each tool** (the `/mcp` route is one
endpoint):

| Scope | Grants |
|---|---|
| `catalog:read` | every read tool, `verify_seal`, `list_actuals` |
| `estimate:run` | all 7 compute/export/import tools |
| `actuals:write` | `log_actuals` only |
| `norms:contribute` | `contribute_norm` / `list_contributions` only — grants no read, no approval |

The web client deliberately holds only `catalog:read` — a person can calculate but never
save. Writes belong to machine clients. **Never widen a scope to make something work.**

## Session flow (raw client recipe)

Stateless — no session id, no `notifications/initialized` required. Call `initialize` once to
read the `instructions` string, then one-shot every `tools/call`.

```bash
BASE="https://resources.hilir.app/mcp"
TOKEN="…"   # Access JWT or estate key
H=(-H "Authorization: Bearer $TOKEN" -H "content-type: application/json")

# optional: read the built-in instructions (version + counts + the loop + the rules)
curl -fsS "${H[@]}" -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"agent","version":"1.0"}}}' "$BASE"

# one-shot tool call — works with no session
curl -fsS "${H[@]}" -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_catalog_info","arguments":{}}}' "$BASE"
```

## The 39 tools

**32 always present** — 25 read (`catalog:read`) + 7 compute (`estimate:run`) — plus 5
store-bound tools that appear in `tools/list` **only where their store exists** (D1 on the
deployed Worker, JSONL for stdio). No store ⇒ the tool is absent, not silently broken.

### Read (`catalog:read`)

| Tool | Purpose |
|---|---|
| `get_catalog_info` | **Start here.** Version, current year, counts per class/array (**incl. 3,155 raw facts**), tool map, the rules. No records, small response |
| `list_classes` | The 13 declared classes (Man … Money, Escalation) with record counts |
| `list_vocabularies` | **Call before you filter.** Every exact filter value accepted, with counts (`{"class":"money"}`, `{"q":"pipe"}` narrow it); `{"class":"facts"}` gives facts.sources/items/kawasans/years/dates/projects/units/statuses |
| `search_catalog` | One search across every array, incl. raw source facts (`class:'facts'`); class/status filters; complete `total_matches`, capped page |
| `get_record` | Full record by id from **any** array, incl. `FACT-MYJRATOL-*` / `FACT-CONS-*`; a miss returns `suggestions`, never an inferred record |
| `search_activities` | Norms by keyword/discipline/phase/sector/source/status; pageable (`next_offset: null` = last page) |
| `get_norm` | Full norm record: crew, output, applicable factors, provenance |
| `get_cost_norms` | Money unit rates; filter by region/country/**year**/currency/status; paged, max 50 |
| `list_escalation_indices` | Index list by currency/location; paged; omits `applies_to` — `get_record` on the index for that |
| `escalate_cost` | Move a Money figure between years through a **named** index (`index_id` required, never auto-picked); returns `index_status` + a seal |
| `get_cost_baseline` | Stored rates re-expressed at one target year by walking indices — derived and labelled; no index ⇒ unchanged with a note |
| `get_labour_rates` | Published gross **monthly** wages by occupation, in the source currency — no hours basis stored; hourly is the caller's explicit conversion |
| `get_material_prices` | Prices per material unit (cement per bag, steel per tonne); **no quantity, no work item** — combine with consumption from your drawings |
| `get_equipment_rates` / `get_margins` | Declared classes with **zero records** — honest empty answer + an acquisition plan; no figure is invented to fill them |
| `get_consumption_coefficients` | **27 sourced consumption facts** (`FACT-CONS-*`: kg/m³ cement/sand/aggregate, L/m² bitumen, kg/m² BRC, spacing→count/m); filter by `material`, paged; derived-from-spec values are low confidence. Honest `no_source` when a filter matches nothing |
| `list_factors` | Productivity/place factor library; place tables carry state/district; `applicable_norms` warns when nobody accepts the factor (fails closed) |
| `list_sources` / `get_benchmarks` / `recommend_sources` / `compare_sources` | Source registry; top-down benchmarks; precedence chain for a project profile; triangulation view for one norm |
| `list_trades` / `list_equipment` / `list_wbs_templates` | Crew trade, machine and WBS vocabularies with example norm ids |
| `map_boq_to_wbs` | Line-level BOQ → WBS by exact/key synonym rules + unit agreement; every unmatched line gets a reason; `mapped + unmatched === lines.length` |
| `verify_seal` | Recompute an estimate from its inputs and compare the seal — prove before you hand over |
| `list_actuals` | Raw logged actuals (audit a calibration ratio or confirm a write landed) |
| `get_calibration` | Per-norm actual-vs-norm ratios — informational, never an automatic edit |

### Compute (`estimate:run`)

| Tool | Purpose |
|---|---|
| `estimate_resources` | The core call: norm × quantity × factors → MH ranges, crew, equipment, duration, seal, basis, warnings. Errors come back as JSON, not exceptions |
| `suggest_wbs` / `get_logic_template` | WBS template (paged norms per discipline); FS logic chains with lag days |
| `build_schedule_block` | Estimate activities, apply FS links, aggregate crew/equipment — use `peak_qty` (not flat `total_qty`) for site staffing |
| `export_schedule` | `msp_xml` / `p6_xml`; durations are absolute hours per each norm's own `hours_per_day` (or shift basis); calendar presets: `petrochemical_2_shift`, `malaysia_civil_construction`, or a custom rota |
| `export_basis_of_estimate` | Markdown BoE — the reviewable, signable artifact; carries a `HUMAN REVIEW REQUIRED` marker when a restricted source is inside |
| `import_schedule` | Read a P6 XER / P6 XML / MSP XML file **back** and compare durations vs norms. Quantities are never inferred — unmapped activities are returned in `unmatched` with a reason, never matched by name similarity |
| `parse_schedule` | Parse a P6 XML/XER or MSP XML file into an **`atur.schedule/1`** network (links, summaries, only printed constraints/cost/actuals); no holidays or ranges invented — atur owns CPM, risk, EVM and the planning calendar |

### Store-bound (appear only where the store exists)

| Tool | Scope | Purpose |
|---|---|---|
| `log_actuals` | `actuals:write` | Log an actual; `unit`/`norm_version` derived from the norm, never taken from the caller |
| `contribute_norm` | `norms:contribute` | Submit a proposed norm — **upload and save only**, status always `submitted`; nothing enters the catalog; `reference` (page/section/URL) is mandatory — no evidence, no save |
| `list_contributions` | `norms:contribute` | Readback of the raw queue; same scope as the write, no approval path |

## The agent loop

```
get_catalog_info          → what exists, where, and the current version/counts
list_vocabularies         → the exact filter values, before you filter
search_catalog / search_activities / get_cost_norms   → find the record
get_record / get_norm     → the full record
recommend_sources         → precedence chain for the project profile
estimate_resources        → compute
verify_seal               → prove it before you hand it over
build_schedule_block → export_schedule(format) → export_basis_of_estimate
import_schedule           → read a real P6/MSP file back and compare
(after execution) log_actuals → list_actuals / get_calibration
(new evidence) contribute_norm → admin review, outside the API
```

An unknown id is never guessed: `get_record` returns `suggestions`.

## Filter honesty — the most common failure mode

Filters match **exact** ids/keys. A guessed spelling (`piping` for `piping_mechanical`,
`Malaysia` for `malaysia`) returns an **empty page that looks like missing data**. Always call
`list_vocabularies` first; when an empty result surprises you, re-check the vocabulary rather
than concluding the catalog lacks the topic. A list is absent when it has no values — absent
≠ empty array.

Paging: `offset`/`limit` (max 50 on `get_cost_norms`, `list_escalation_indices`); `next_offset`
is `null` on the last page — that is how a pager knows to stop. Counts are complete even when
the returned page is capped.

## The rules that are the product

- **Never invent a number.** Empty results are honest answers, not invitations to approximate.
- **Never convert currency.** `escalate_cost` refuses cross-currency; labour rates carry no
  hours basis — an hourly rate is your explicit, declared conversion.
- **Never escalate without a named index** from `list_escalation_indices`. Refusals are
  explicit (cross-currency, year outside the index window, wrong source year).
- **Draft indices are usable by default** — every escalation result carries `index_status`;
  pass `exclude_draft: true` only when the caller wants reviewed factors.
- **Always carry `seal` + `catalog_version` into anything you produce.** An estimate without
  them cannot be audited or replayed. If a record's `status` is `draft`, say so.
- **`license: restricted` records are internal use only**, and every one carries
  `provenance.publication_review: human_required`: a named human reviews before any **external**
  publication (BOE, report, export, web). Internal MCP/REST reads are unaffected. Restricted
  does not mean "never stored" — it means internal-use plus the external-publication gate.
  The exported schedule XML does not carry the flag — the human gate applies before release.
- Write only inside the scopes that grant them; a tool call needing a missing scope returns
  `isError: true` naming the scope.

## Failure signatures

| Symptom | Cause |
|---|---|
| `401` | no valid principal — expired/revoked key, or none sent |
| `403 {"error":"forbidden","required":"<scope>"}` | valid credential lacking that scope — expected behaviour, not a bug |
| `isError: true` naming a scope | same, from inside a tool (route cannot gate per tool) |
| Empty search/filter page | usually a guessed vocabulary key — check `list_vocabularies` |
| `503` on actuals write | no actuals store configured — fail closed, nothing was recorded |

## When the MCP layer is unavailable

REST answers the same questions with the same credential, no session:

```bash
curl -fsS -H "Authorization: Bearer $TOKEN" https://resources.hilir.app/api/v1/version
curl -fsS -H "Authorization: Bearer $TOKEN" "https://resources.hilir.app/api/v1/norms?q=weld&limit=20"
curl -fsS -H "Authorization: Bearer $TOKEN" -X POST -d '{"norm_id":"CON-STR-STEEL-ERE","quantity":850}' \
  https://resources.hilir.app/api/v1/estimate
```

Endpoints mirror the tools: `/norms` (+`/{id}`), `/cost-norms`, `/escalation-indices`,
`POST /escalate`, `/cost-baseline`, `/material-prices`, `/labour-rates`, `/factors`,
`/sources`, `/benchmarks`, `/wbs`, `POST /estimate`, `POST /verify` (409 = seal mismatch),
`POST /export/schedule`, `GET`/`POST /contributions`, `GET`/`POST /actuals`, `/whoami`.
For offline work, `GET /download/catalog.sqlite` (and the six JSON chunks) serve the whole
catalog under `catalog:read` — the MCP server itself does not read that file.

**Credential hygiene.** The `Bearer`/`x-api-key` header above goes only to
`resources.hilir.app` (or the local stdio server) — that is the credential's one legitimate
destination. Never read, display, echo or paste the key itself into a response, a log, an
exported BoE or schedule XML, an issue or commit, or any other host; the exports this API
produces carry `seal` + `catalog_version`, never a credential. Use the key the user or
environment has configured — never a guessed or invented one. If a key leaks, report it and
let the owner mint a replacement in hilir Access; do not rotate credentials yourself.

## Numbers you must not quote from memory

Counts, versions and statuses change with every ingest. The figures in this skill's
description were true at writing (catalog 0.36.0+fbea1480be14, 39 tools) — always read the
current truth from `get_catalog_info` / `get_catalog_info.status_mix`, never from memory,
and never assume an array is approved.
