# MCP audit — hilir-skills

This is the document of record for MCP-server audits of the nine skills. Future
audits update this file in place; do not add dated copies.

**Audited: 2026-11-26.**

## How this was checked

- **Live calls** through this session's mounted MCP connections (the
  dev-hilir-libraries plugin 0.4.4, which mounts the seven `*.hilir.app` servers,
  plus the standalone conteng mount) — one cheap read tool per server.
- **Code registries** grepped in each app repo (read-only).
- **Canonical `SKILL.md`** counts and tool names vs the live toolsets.
- **Mirror sync**: canonical vs `~/.agents/skills`, `~/.claude/skills`,
  `~/.zcode/skills`, the plugin cache, **and each app repo's own copies**
  (`<app>/.claude/skills`, `<app>/.github/skills`). Those last two were not
  checked in the first pass and three of the four stale `dwsim-mcp` copies
  were hiding there (2026-11-26).
- **Unauthenticated production probes** of all nine endpoints.

No credentials were printed or stored; there is no estate key in this shell, so
no authenticated production `tools/list` — live counts come from the session's
own mounted connections.

## Result table

| Skill | Server | Connected this session | Live evidence | Live tools | Code registry | Canonical skill | Verdict |
|---|---|---|---|---|---|---|---|
| `atur` | `atur.hilir.app/mcp` | yes | `get_capabilities` → 27 tools + scopes | **27** | 27 (`src/tools.ts`) | 27 stated; 27 named (the table listed 26 — `audit_log` was missing; fixed 2026-11-26) | ✅ in sync |
| `conteng` | `conteng.hilir.app/mcp` | yes | `conteng_files` → 3 stored files | **31** | 32 (`app/tools/__init__.py`, incl. `conteng_grid`) | “25 tools”; 24 named; says DOCX/XLSX/PPTX unsupported | ❌ stale both directions |
| `dwsim-mcp` | `dwsim.hilir.app/mcp` | yes | `dwsim_unitop_list_types` → 45 types | **70** | 70 (`dwsim-fork`) | 70 (58 upstream + 12 HILIR) | ✅ in sync |
| `hilir-cad` | `cad.hilir.app/mcp` | yes | `drawing_list` → 7 drawings | **34** | `TOOL_COUNT = 34` (test-enforced) | 34 | ✅ in sync |
| `hilir-thunderbolt` | `thunderbolt.hilir.app/mcp` | yes | `get_capabilities` → engine 0.2.3 | **32** | 27 engine + 5 project | 32 (27 + 5) | ✅ in sync |
| `mygis-mcp` | `mygis.hilir.app/mcp` | yes | `layers_list` → 27 layers; `service_status` | **32** | 32 (`src/tools/*.ts`) | 32 | ✅ in sync |
| `techlib-mcp` | `techlib-api.hilir.app/mcp` | yes | `library_stats` → 2369 sources … | **32** | 32 (`agent/server.py`) | “26 tools” | ❌ skill stale by 6 |
| `tlea-mcp` | `tlea-api.hilir.app/mcp` | yes | `stats` → last_load 2026-11-25 | **9** | 9 (`api/mcp.py`) | 9 | ✅ in sync |
| `resources-mcp` | `resources.hilir.app/mcp` | **no** — configured in the plugin `.mcp.json` but not mounted | unauth probe → 401 (alive, gated) | — | 39 (`server/src/tools.ts` `TOOL_MAP`) | 39 | ⚠️ endpoint up; code = doc; session mount failed |

**Production endpoints:** all nine alive. Open `/healthz` on cad
(`{"status":"ok","tools":34,"auth":true}`) and thunderbolt (`0.2.0-p1`);
auth-gated `/healthz` (401) on conteng, techlib-api, tlea-api; no `/healthz`
route on atur/dwsim/mygis/resources but `/mcp` answers 401 — alive and gated.
No `sk-` keys exist anywhere (docs and environment both confirm).

**Totals:** 267 tools exposed across the session's 8 live connections
(27+31+70+34+32+32+32+9). Across all nine servers (resources at its registry
count of 39) the estate serves **306** tools today (267 + 39); the plugin
README's table still says 258 (verified 2026-11-23). *Corrected 2026-11-26 —
the first draft printed 275, which its own two figures do not produce.*

## Findings

1. **Eight of nine servers verified live end-to-end.** Every mounted server
   answered a real read call with current data — not just a health probe. The
   one gap is resources (below).

2. **`hilir-resources` is configured but did not connect in this session.** The
   plugin `.mcp.json` lists
   `hilir-resources → https://resources.hilir.app/mcp` alongside the seven that
   mounted, yet no resources tools are exposed. The endpoint itself is alive
   (401 unauth), and the atur `get_capabilities` companion block records it
   “live and verified (2026-11-23)”, catalog 0.36.0. Likely cause: the
   session's token lacks the resources scopes, or the mount was rejected at
   startup — check the ZCode MCP connection error for this server.

3. **The canonical conteng skill contradicts the live server in both
   directions** — the biggest documentation gap:
   - claims **“25 tools”**; the live server exposes **31**, the code registry
     has **32**;
   - claims **“DOCX/XLSX/PPTX are unsupported (phase 2)”** — but
     `conteng_create_docx`, `conteng_create_xlsx`, `conteng_create_pptx` (plus
     `conteng_create_pdf`, `conteng_create_image`, `conteng_edit_pdf`,
     `conteng_edit_image`, `conteng_remove_annotation`) are all live. An agent
     following the skill would refuse Word/Excel/PowerPoint deliverables the
     server can actually produce;
   - documents `conteng_grid` as an available tool, but the session's server
     does not expose it — the code tip has it (commit 768009e, “32 tools”,
     2026-11-24; `_guard` is only error normalisation, not an auth gate), so
     the deployed image is one tool behind the code tip.
   - Both installed mirrors (`~/.agents/skills`, `~/.zcode/skills`) share the
     same omissions, so every surface currently mis-states this server.

4. **The canonical techlib skill is stale by six tools.** Verified correct at
   26 tools on 2026-11-23; the server has since grown to 32. Undocumented in
   the skill: `get_product`/`list_products` (458 product rows live),
   `get_material`/`list_materials` (249), `get_drawing`/`list_drawings` (345),
   `list_by_tag`. No phantom tools — everything the skill names exists.

5. **Five mirrors are stale; canonical is newer in every case** — re-sync
   `~/.agents/skills` (and `~/.zcode/skills` for conteng) from this repo:
   - `dwsim-mcp`: mirror says 69 (58 + 11) and is missing
     `dwsim_flowsheet_stream_table`; canonical/live = 70 (58 + 12).
     **CLOSED 2026-11-26** — but four copies were stale, not one: `~/.agents`,
     `~/.claude`, and both app-local mirrors `hilir-dwsim/.claude/skills` and
     `hilir-dwsim/.github/skills` all still carried the pre-v8 text (69 tools,
     no stream-table row, and the retired `Authorization: Bearer sk-…`
     wording at three sites). Re-verified before syncing — live
     `dwsim_unitop_list_types` → 45 types, the live mount exposes
     `dwsim_flowsheet_stream_table`, and the fork registry at HEAD `2879294`
     greps 70 dispatch names, so live = code = canonical = 70. All four
     re-copied from canonical, checksum-identical (`e54372d7e6a419f4…`); the
     two app copies committed and pushed in `hilir-dwsim` as `69facc1`;
   - `mygis-mcp`: mirror says 35 POI category presets; canonical/live = 44;
   - `resources-mcp`: mirror's tool table says 32 always-present
     (25 read + 7 compute); canonical = 34 (26 read + 8 compute) + 5
     store-bound = 39, matching the code registry;
   - `techlib-mcp`: mirror is missing the “one document per source url”
     paragraph;
   - `conteng`: canonical recently added the count claim and a “find a handle
     you lost” paragraph the mirrors lack.

   **In sync:** `atur`, `hilir-cad`, `tlea-mcp` (in `~/.agents/skills`) and the
   plugin-cache `hilir-thunderbolt` copy (matches canonical exactly).
   `hilir-thunderbolt` has no `~/.agents`/`~/.zcode` mirror at all — the
   plugin cache copy is its only installed form.

6. **mygis upstream health at audit time** (`service_status`, 2026-11-26
   08:55 UTC): **Overpass is entirely down** — all three mirrors timed out, so
   `poi_search`, `whats_here`, `features_in_area` and live `road_access`
   queries fail right now (`road_access` falls back to the
   `osm-roads-malaysia` snapshot; POI queries have no snapshot equivalent).
   FMRS state services answer for 4/12 states (Johor, Perlis, Selangor,
   Terengganu); the national HSK overlay is token-gated; OSRM, PLANMalaysia
   gisdev, JUPEM UPI and JPPH MyGOS all answer.

7. **The plugin README's tool table is stale** (the ZCode Plugin repo —
   noted here, not edited from this repo): its counts date to 2026-11-23
   (techlib 26, resources 37, dwsim 69, cad 30, thunderbolt 31, atur 24 =
   “258 across the eight”), against 275 today. The README itself warns that
   counts drift and to re-check with each server's own discovery output.

8. **Thunderbolt version split observed:** the session's engine reports 0.2.3
   while the production app `/healthz` reports `0.2.0-p1` (the Worker serving
   the UI and the five project tools, with the engine at
   `thunderbolt-origin.hilir.app`). Two components version separately — not
   necessarily a fault, but worth a look at release hygiene.

9. **TLEA's translation layer is effectively empty:** 0 documents translated,
   2 untranslated (stats; catalogue freshly loaded 2026-11-25 23:22 UTC). The
   termbase and hybrid search work; translated full text does not exist yet.

10. **`atur`'s tool table was one row short.** The skill states its count
    (`## Tools (27)`) and both the live `get_capabilities` and `src/tools.ts`
    expose 27, but the table named only 26 — `audit_log` (read; the write
    history) appeared only in the honesty rules prose. **Fixed 2026-11-26:** the
    row is added, both installed mirrors re-copied, and a name-by-name diff
    against the registry now shows 27 of 27 named with no phantom names. (A
    naive `name: "…"` grep over `src/tools.ts` picks up a data literal
    `"anchor"` inside `delay_event_add` — it is not a 28th tool.) The CAD test
    suite enforces `TOOL_COUNT`/registration lockstep — the pattern the other
    repos could copy.

## Live evidence collected (2026-11-26)

- **atur** `get_capabilities`: 27 tools (including the read-scoped `audit_log`),
  scopes `atur:read`/`atur:write`; `audit_log` answered with 3 writes
  (`project_purge` ×2, `project_create`, 2026-11-24) — project_id, actor, tool,
  action and timestamp per row;
  Gantt + report PDFs served under the caller's sign-in; 5M Resources recorded
  as companion, catalog 0.36.0, “live and verified (2026-11-23)”.
- **conteng** `conteng_files`: 3 stored files (annotated `plan.pdf`, 18 and
  23 Nov) — the annotation store is warm from earlier sessions.
- **dwsim** `dwsim_unitop_list_types`: 45 unit-op types including
  `PIDController`; the 70-tool surface matches the fork grep exactly.
- **cad** `drawing_list`: 7 drawings (5 PFD, 1 SLD, 1 GA). Production
  `/healthz` advertises `tools: 34`.
- **thunderbolt** `get_capabilities`: IEC 60909-0:2016 short circuit,
  IEEE 1584-2018 arc flash, IEC 60255-151 coordination, N-1, D1 project
  storage; honest `not_implemented` list (no DXF, no fuse curves, no N-2).
- **mygis** `layers_list`: 27 self-hosted snapshot layers (flood-risk, slope,
  Melaka zoning/NDCDB lots, OSM roads, BCER/AER/USGS/GEM/StatCan/AAFC …),
  each with licence and extraction date; `service_status` as in finding 6.
- **techlib** `library_stats`: 2369 sources, 253 documents, 13 192 chunks,
  19 729 vectors, 402 technologies, 1342 vendors, 505 equipment classes,
  417 register entries, 1116 projects, 458 products, 345 drawings,
  249 materials.
- **tlea** `stats`: 547 sources, 162 documents, 277 technologies, 51 patents,
  289 vendors, 405 equipment, 170 projects, 169 standards, 9 restricted
  sources; last load 2026-11-25 23:22 UTC.

## Recommended actions

1. **Rewrite `skills/conteng/SKILL.md` against the live 31-tool surface**:
   document the five producers (`create_pdf`, `create_image`, `create_docx`,
   `create_pptx`, `create_xlsx`) plus `edit_pdf`, `edit_image` and
   `remove_annotation`; drop the “unsupported (phase 2)” line; mark
   `conteng_grid` as not-yet-deployed (or deploy the 32-tool image, then
   document it as live).
2. **Rewrite `skills/techlib-mcp/SKILL.md` for the 32-tool surface**
   (products, materials, drawings, `list_by_tag`).
3. **Re-sync the stale mirrors** from this repo (conteng → `~/.agents` +
   `~/.zcode`; mygis-mcp, resources-mcp, techlib-mcp → `~/.agents`).
   `dwsim-mcp` is **done** (2026-11-26): four surfaces, `~/.agents` +
   `~/.claude` + the two `hilir-dwsim` app mirrors — count mirror surfaces,
   not one directory: the app repos carry their own copies and this audit's
   first pass missed three of the four.
4. **Investigate the `hilir-resources` mount failure** — token scopes vs
   server auth.
5. **Update the plugin README tool table** in the ZCode Plugin repo (edited
   there, not from this repo).
6. **mygis**: the Overpass outage is transient — retry before assuming data
   loss; only `road_access` has a snapshot fallback.
7. **Give every server an unauthenticated `/healthz` advertising its tool
   count** (cad already does) — it reduces this audit to one curl per server.
