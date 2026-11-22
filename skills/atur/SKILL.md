---
name: atur
description: Use the hilir Atur MCP server (atur.hilir.app) — the project planner and schedule record for construction and oil & gas projects: run CPM (dates, float, critical path), DCMA 14-point health with out-of-sequence and float-erosion checks, three-point Monte Carlo risk (P50/P80), earned value with S-curve and cash flow, delay window analysis / time impact, a 3/6-week look-ahead, and the Malaysian working calendar and approval lead times — while storing projects, append-only revisions, signed baselines, weekly progress with work-package roll-up, delay events and an audit of every write. 5M Resources supplies the data (norms, Malaysian calendars, rates, BOQ rules, P6/MSP parsing). Use when asked to build or check a schedule, baseline a programme, assess float or schedule health, run a risk analysis for a P80 date, report earned value, record weekly progress, analyse a delay or EOT, or answer what changed and who approved it.
---

# Atur MCP (planner + schedule record)

**atur plans; 5M Resources supplies data.** atur computes the schedule — CPM, DCMA health, risk,
earned value, delay analysis, look-ahead — holds the Malaysian planning data it needs (working
calendar, approval lead-time ranges), and keeps the auditable record: projects, append-only
revisions, baselines a named human signed off, progress, delays, analyses and an audit. Resources
gives it norms that produce durations, rates, BOQ→WBS rules and parsed P6/MSP files — two tools,
`parse_schedule` and `map_boq_to_wbs`. Contract forms belong to **Hilir Kontrak**; atur records a
clause label and never asserts a clause number. Never re-derive data at atur, and never pretend a
missing data tool exists.

## Connection

| | |
|---|---|
| MCP endpoint | `POST https://atur.hilir.app/mcp` |
| Discovery | `GET https://atur.hilir.app/.well-known/oauth-protected-resource` |
| Auth | `Authorization: Bearer <credential>` — `x-api-key` accepted identically |
| Credential | hilir Access JWT with `aud = https://atur.hilir.app`, or a person's estate key |
| Scopes | `atur:read` (every read and analysis), `atur:write` (create, import, freeze, progress, delays, analyses) |
| Transport | Streamable HTTP, stateless — no session handshake |

```bash
TOKEN="…"
H=(-H "Authorization: Bearer $TOKEN" -H "accept: application/json, text/event-stream" \
   -H "content-type: application/json")
curl -fsS "${H[@]}" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' https://atur.hilir.app/mcp
curl -fsS "${H[@]}" -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_capabilities","arguments":{}}}' https://atur.hilir.app/mcp
```

A 401 means missing/expired/wrong audience; a 403 on a write means the credential lacks
`atur:write`. Refresh the credential; never widen a scope to make a call work.

## The loop

```
5M Resources (data)                     atur (plan + record)
───────────────────                     ────────────────────
parse_schedule (P6/XER/MSP) ──────────▶ project_create → schedule_import (preview → apply)
estimate_resources (durations) ────────▶
atur calendar_get / localisation_lookup ─▶ (state weekends, holidays, monsoon, approval ranges)
                                        schedule_cpm
                                        schedule_health  ← the baseline gate
                                        schedule_sra     → contingency, P80
                                        baseline_freeze (sign-off, health gate)
log_actuals ◀──────── calibration_export
                                        progress_ingest (weekly, work-package %)
                                        schedule_diff (format=markdown) → the weekly report
                                        schedule_evm · schedule_lookahead · eot_analyse
```

## Tools (24)

| Tool | Scope | What it does |
|---|---|---|
| `whoami`, `get_capabilities` | — | identity; the exact planner/data boundary and what is not implemented |
| `calendar_get` | read | Malaysian working calendar: state weekend rule, holidays by confidence, monsoon/earthwork windows |
| `localisation_lookup` | read | approval lead-time ranges (DOE, BOMBA, TNB, councils) with basis and `as_of` |
| `project_create`\* | write | create a project |
| `project_list`, `project_open` | read | projects; a revision's full document + baseline + delays + progress history |
| `project_archive`\* | write | soft delete; revisions survive |
| `revision_list` | read | history: kind, note, author, data date, task count |
| `schedule_import`\* | write | validate + store a revision; `preview: true` diffs without writing |
| `schedule_diff` | read | structural diff; `format: "markdown"` gives the weekly report; defaults to baseline → latest |
| `schedule_cpm` | read | early/late dates, total/free float, critical path, summary roll-up |
| `schedule_health` | read | DCMA 14 + out-of-sequence, float erosion, WBS completeness; `na` with reasons |
| `schedule_sra` | read | seeded Monte Carlo: P10/P50/P80/P90/P95, contingency, criticality, sensitivity |
| `schedule_evm` | read | BAC/PV/EV/AC, SPI/CPI, EAC/ETC/VAC/TCPI, S-curve, cash flow |
| `schedule_lookahead` | read | 3/6-week look-ahead with open predecessors |
| `eot_analyse` | read | window analysis / TIA: days moved per window, concurrency flagged |
| `baseline_freeze`\* | write | freeze a revision: named sign-off, health gate, reasoned rebaseline |
| `progress_ingest`\* | write | per-task actuals → a new revision, with work-package roll-up |
| `calibration_export` | read | actual man-hours as rows for resources' `log_actuals` |
| `delay_event_add`\* | write | record a delay event |
| `record_analysis`, `analysis_list` | write / read | bind an analysis result to the revision it ran on |

## The document — `atur.schedule/1`

```json
{
  "schema": "atur.schedule/1",
  "name": "Pengerang Tank Farm — Phase 2",
  "data_date": "2026-11-22",
  "currency": "MYR",
  "calendar": { "hours_per_day": 8, "weekend": [0, 6], "holidays": ["2026-12-25"],
                "state": "JHR", "source": "resources get_work_calendar",
                "exclude_windows": [{ "from": "2027-01-01", "to": "2027-01-31", "reason": "NE monsoon" }] },
  "tasks": [
    { "id": "S4000", "name": "Civil", "type": "summary", "duration_days": 0 },
    { "id": "A1020", "name": "Pile caps", "wbs": "4000", "parent": "S4000", "type": "task",
      "duration_days": 12,
      "predecessors": [{ "id": "A1010", "type": "FS", "lag_days": 0 }],
      "constraint": { "type": "SNET", "date": "2027-01-05" },
      "three_point": { "min": 10, "most_likely": 12, "max": 18 },
      "cost": { "budget": 184000 },
      "actual": { "start": "2027-01-06", "finish": null, "pct": 35, "cost": 61000, "manhours": 940 },
      "planned": { "start": "2027-01-05", "finish": "2027-01-20" },
      "boq": { "item": "2.4.1", "unit": "m3", "qty": 340 },
      "source": { "norm_id": "CIV-PILECAP-01", "norm_version": 3 } }
  ],
  "delays": [{ "id": "D1", "name": "Late AFC drawing", "start": "2027-02-01", "end": "2027-02-12",
               "responsible": "employer", "contract_clause": "PAM 2018 cl. 23.1", "affected_tasks": ["A1020"] }]
}
```

Rules that matter:

- `id` stable across revisions; never renumber.
- `duration_days` and `lag_days` are **working days** on the document calendar; negative lags are
  preserved (the health check flags them).
- `parent` is what lets atur roll summary/WBS dates up; `type` is task | milestone | summary.
- Missing values are omitted — an unknown cost is not a free activity.
- `three_point` is required for risk; atur will not invent a range from one duration.
- `actual.manhours` is what `calibration_export` hands back to the norm catalogue.

## Analysis semantics you must repeat to a user

- **CPM**: FS successor starts the next working day after its predecessor (lag 0 = adjacency).
  Total float = LS − ES in working days; free float is computed for FS links only. Float ≤ 0 is
  critical. Constraints are clamps.
- **Health**: DCMA thresholds are 5 % (0 % for leads and negative float), CPLI/BEI ≥ 0.95 — they
  are conventions, not law. `na` means the data is absent, not that the check passed. `extra`
  checks (out-of-sequence, float erosion, WBS completeness) are atur's, not DCMA's.
- **Risk**: duration-only, activities independent, triangular distribution, seeded. Report the
  count of activities with no three-point estimate; they were treated as certain.
- **EVM**: `na` without budgets; `spi`/`cpi` null when a denominator is zero; EAC = BAC/CPI.
- **EOT**: days moved per window with concurrency flagged. Analysis only — entitlement and clause
  numbers belong to the contract.

## Authority — what an agent may do

| Action | Authority |
|---|---|
| Create a project, import a revision, preview a diff | **Freely** — but preview before every import and show the user |
| Record progress, delay events, analyses | **Freely** — it is the record of what was reported |
| Freeze or supersede a baseline | **Human sign-off required**: `signoff_name` must be a real person who approved it; a rebaseline needs `reason` |
| Override a failed health gate | **Human decision**, with `allow_unhealthy: true` and a `reason`; the override is audited |
| Invent a sign-off, a duration, a cost, a float or a clause | **Never** |

## Recipes

**New project from a planner's file**
1. `resources.parse_schedule({ content })` + `atur.calendar_get({ state: "JHR", year: 2027 })` →
   the document and the calendar it should be computed on.
2. `atur.project_create({ name, currency: "MYR" })`.
3. `atur.schedule_import({ project_id, schedule, preview: true })` → show validation + diff.
4. `atur.schedule_import({ project_id, schedule })` → revision 1.
5. `atur.schedule_cpm` and `atur.schedule_health` → show dates and the failing checks.
6. `atur.schedule_sra({ iterations: 1000, seed: 42 })` → P80 and contingency.
7. On a human's approval: `atur.baseline_freeze({ project_id, revision: 1, label, signoff_name })`.
   If health fails, fix it or get an explicit override.

**Weekly update**
1. `atur.progress_ingest({ project_id, period_end, data_date, updates: [{ task_id, pct, actual_start, actual_finish, actual_cost, actual_manhours }] })`.
2. `atur.schedule_diff({ project_id, format: "markdown" })` → the weekly report.
3. `atur.schedule_evm({ project_id })` → SPI/CPI, EAC, curve.
4. `atur.schedule_lookahead({ project_id, weeks: 6 })` → ready-to-start versus blocked.
5. `atur.calibration_export({ project_id })` → post the rows to `resources.log_actuals` (actuals:write).

**Delay / EOT**
1. `atur.delay_event_add({ project_id, event: { name, start, end, responsible, contract_clause, affected_tasks } })`.
2. `atur.eot_analyse({ project_id, contract_form: "PAM_2018", method: "window" })`.
3. Report days moved per window and concurrency; say plainly it is analysis, not entitlement.
4. `atur.record_analysis({ project_id, revision, kind: "eot", result, engine: "atur" })` if the
   result must be preserved against that revision.

**Rebaseline**
`atur.baseline_freeze({ project_id, revision, label, signoff_name, reason })` — the reason is
mandatory when a baseline exists; the old one is superseded, never edited.

## Honesty rules and error codes

- Only a human freezes a baseline; if you do not have a person's approval, stop and ask.
- `preview` writes nothing. Report `written: true` and the `revision` number only when they appear.
- A skipped progress line is a finding: report the task id and reason.
- `schedule_health` `na` is not a pass; say which data is missing.
- Calendar data carries confidence: fixed-date federal holidays are gazetted, lunar/Islamic dates
  are **estimated** until the gazette. Quote the confidence and the verify note when a date matters.
- Analysis is bound to a revision; quoting it against another revision is a misquote.
- If `audit_log` does not show a write, it did not happen through atur.

Codes: `forbidden` (scope), `not_found`, `schedule_invalid` (validation in the message),
`reason_required`, `health_gate` (fix or override), `nothing_applied`, `no_comparison_base`,
`logic_cycle`, `no_revisions`.

## What atur does not do

Norm catalogues, cost rates, BOQ→WBS rules and file parsing (5M Resources); contract forms and
clause text (Hilir Kontrak); site constraints (mygis); resource levelling and crew histograms;
correlated risk; weather simulation beyond explicit exclusion windows; legal advice.
