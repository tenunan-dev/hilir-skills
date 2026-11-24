# hilir-skills

Canonical home for Hilir agent skills. One subfolder per skill, each containing a
`SKILL.md` in the portable [Agent Skills](https://agentskills.io) format that
Claude Code, GitHub Copilot and ZCode all read:

```
skills/<skill-name>/SKILL.md
```

## Rules

- **Frontmatter carries exactly `name` and `description`.** `name` matches the
  subfolder (lowercase, hyphens). `description` is third-person, states what the
  skill serves and ends with "use when …" triggers — it is the only text an
  agent sees when deciding to load the skill, so triggers matter more than
  completeness. Everything else lives in the body (progressive disclosure).
- **The copy here is canonical.** App-local copies (`~/.agents/skills/…`,
  `~/.zcode/skills/…`, a plugin's `skills/…`, `.github/skills/…` in an app
  repo) are synced mirrors. Fix a skill by PRing here, then re-sync — do not
  fork the content in an app.
- **One skill = one folder.** A skill that outgrows a single file may add
  sibling files (e.g. `examples.md`, a small CSV) and reference them with
  relative paths from `SKILL.md`. Do not split a skill across several folders.
- **Server-backed skills name their MCP server** in the body (e.g.
  `hilir-thunderbolt`) and the tool names exactly as the server exposes them.
  The skill must be verified against the server's own capability output, not
  written from memory.

## Current skills

| Folder | Serves | MCP server | Canonical? |
|---|---|---|---|
| `skills/hilir-thunderbolt` | Electrical power-systems studies (IEC 60909, load flow, arc flash, coordination, N-1, project storage) | `thunderbolt.hilir.app/mcp` | yes |
| `skills/conteng` | PDF/image annotation at mm precision (review pins, callouts, redaction, drawing CRS, exports, review reports) | `conteng.hilir.app/mcp` | yes |
| `skills/hilir-cad` | Drawing generation via the JELAS CAD MCP server (PFD/P&ID/plan/GA/plot/iso/SLD, solids, MTO, issued A3/A1 sheets with sheet sets and an H&MB stream table, revision diff, DXF/DEXPI/STEP/STL) | `cad.hilir.app/mcp` | yes |
| `skills/dwsim-mcp` | Chemical process simulation via the HILIR DWSIM MCP server (steady-state + dynamic flowsheets, reactions, PID tuning, pump curves with NPSHr, pipe hydraulics, shell-and-tube rating, distillation screening, scenario sweeps) | `dwsim.hilir.app/mcp` | yes |
| `skills/techlib-mcp` | Oil/gas/petrochemical/energy technology evidence via the HILIR Technology Library (technology cards, costs with basis, vendors, equipment classes, plants and projects, source documents, standards, glossary, Postgres/graph layer) | `techlib-api.hilir.app/mcp` | yes |
| `skills/tlea-mcp` | East Asian energy-technology evidence via TLEA (hybrid search with zh/ja/ko termbase resolution, per-kind filter contract, licence/reuse labels) | `tlea-api.hilir.app/mcp` | yes |
| `skills/mygis-mcp` | Malaysia GIS via mygis (geocoding + admin/cadastral chain, land cover/use, forest and Sarawak overlays, terrain/bathymetry, routing, static/Wayback maps, self-hosted snapshot layers, construction site lens, site_report aggregate) | `mygis.hilir.app/mcp` | yes |
| `skills/atur` | Project planning and the schedule record via Atur (CPM, DCMA 14-point health with OOS/float-erosion extras, SRA Monte Carlo, EVM/S-curve, EOT delay analysis, look-ahead, schedule diff, BOQ-vs-programme reconciliation, baseline freeze, test-project purge, Malaysian calendar data, and Gantt + report PDFs in the P6/MS-Project view) | `atur.hilir.app/mcp` | yes |
| `skills/resources-mcp` | 5M Resources cost/man-hour catalog (man-hour norms, Money-class cost norms, escalation indices, material prices, labour rates, benchmarks) with content-hash verification and estimate seals | `resources.hilir.app/mcp` | yes |

## Planned to migrate here

| Folder | Serves | Currently canonical at |
|---|---|---|
| `skills/hilir-libraries` | Routing across the eight HILIR libraries (incl. Atur) | `ZCode Plugin/hilir-libraries/skills/hilir-libraries/` |

## How to install a skill from here into an app

| Surface | Where the copy goes |
|---|---|
| Claude Code / ZCode (personal) | `~/.agents/skills/<name>/` (or `~/.zcode/skills/<name>/`) |
| Claude Code / ZCode (plugin) | `<plugin>/skills/<name>/` — bump the plugin version after sync |
| GitHub Copilot (per app) | `<app-repo>/.github/skills/<name>/SKILL.md` |
| Any other Agent-Skills host | Its own skill directory, same folder name |

Because the format is the portable Agent Skills scheme (`name` + `description`
frontmatter only), the same file works unchanged everywhere — the app decides
where to put it, never how to rewrite it.

## Aligning other Hilir apps

Other Hilir repos that ship or consume skills should:

1. Follow the same `skills/<name>/SKILL.md` layout for any skill they carry.
2. Keep the frontmatter contract above (`name` + `description` only).
3. Treat this repo as the source of truth — add a sync step (or a note in their
   README pointing at this repo) instead of maintaining a divergent copy.
4. When adding a new skill, add its row to the table above.
