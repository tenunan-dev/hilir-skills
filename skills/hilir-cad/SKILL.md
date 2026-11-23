---
name: hilir-cad
description: Generate engineering drawings and 3D solids through the JELAS CAD MCP server (cad.hilir.app) — PFDs, P&IDs, site plot plans, building plans/GAs, piping isometrics, electrical single-line diagrams and parametric solids, with deterministic material take-offs delivered as SVG/PNG/PDF, DXF, DEXPI or STEP/STL. Use it whenever the user asks to draw, sketch, generate, revise or export a process flow diagram, P&ID, plot plan, layout, general arrangement, isometric or SLD; asks for a material take-off (MTO), line list or equipment/door/column schedule; asks for DXF, DEXPI, STEP or STL output; or wants a parametric 3D part. Do NOT use it for electrical analysis (Thunderbolt owns load-flow/short-circuit) or flowsheet simulation (DWSIM) — this service draws and counts, it does not solve those.
---

# JELAS CAD — drawing generation over MCP

`cad.hilir.app` is a headless drafting service: you edit a **validated drawing document**;
the engines (pyDEXPI, ezdxf, build123d) produce the deliverables. You never draw pixels —
you place elements and connect them. The authoritative deep reference is the server's own
guide — **read https://cad.hilir.app/agent-guide before a complex drawing**; this skill is
the working summary. Every tool is self-describing: trust the tool description over memory.

## Connect

```
URL:      https://cad.hilir.app/mcp        (streamable HTTP; initialize issues an
                                            Mcp-Session-Id every later call must send)
Headers:  Authorization: Bearer <credential>
```

The credential is an **estate credential** from access.hilir.app (OAuth token or agent
key), scope `cad:generate`. In this workspace it is normally already connected via the
**hilir-libraries plugin** — the tools appear prefixed
`mcp__plugin_hilir-libraries_hilir-cad__…` (after a cad redeploy the session may need a
MCP reconnect; see the plugin's diagnosing-mcp flow). A 401 = credential missing/invalid;
403 = valid credential but missing scope — fix the grant, not the token.

The initialize response carries server `instructions` — the at-connection summary of this
same loop (symbol_library first, valve-`"through"` discipline, type matrix, MTO, outputs
hygiene, honest limits); read it before your first tool call. This skill and
`GET /agent-guide` carry the deep detail on top of it.

## The core loop — always close it

`symbol_library(type) → drawing_create → element_add / line_add (run_add for iso) → drawing_validate → drawing_render / drawing_view → drawing_export`

1. **`symbol_library(type)` first, every type.** Allowed kinds, required fields and
   nozzle keys are not guessable — `tank` N1/N2, `pump` S/D, `heat_exchanger` SI/SO/TI/TO,
   `column` TOP/FEED/BOTTOM, valves I/O, instruments none.
2. **Place on a grid you choose.** P&ID/diagram units vs civil/plot/iso metres — they do
   not convert. Flow left→right, one row per process line, ≥80 units between equipment
   (symbols are ~60–140 units wide), instruments 60–70 above/below their equipment.
3. **Validate after every batch** (`drawing_validate`). Render/export **refuse** invalid
   drawings — that is deliberate, not a bug.
4. **Deliver:** `drawing_render` (svg/png/pdf, content inlined while the reply stays
   under 64 KiB) or `drawing_view` (PNG as an image in the conversation) for the user to
   see; `drawing_export` for the engineering format (DEXPI for pfd/pid, DXF for
   plan/ga/plot/iso/sld).

## Working discipline

- **Valves are placed as elements but mounted on lines.** The valve sits at its drawn
  position; the line passing through it lists it in `"through": ["v1"]`. A valve is
  **never** a line endpoint — write-time rejection, live.
- **Signals are lines with `"kind": "signal"`** — dashed, no nozzle needed; endpoints
  are element refs (or points). Use for instrument → valve/controller
  (e.g. `LIC-101 → LCV-101`).
- **Each nozzle carries one line only.** Nozzle keys come from `symbol_library`, not you.
- **Always supply `size` and `spec` on lines/runs** — they feed the MTO line-list columns,
  not just the sheet.
- **Errors carry the problem list** (`isError: true`). Fix the document, re-call. Never
  work around a rejection.
- **Revision workflow:** `drawing_open` → `element_update`/`line_add`/`run_update` →
  `drawing_validate` → `drawing_render` + `drawing_export`. `drawing_duplicate` for a
  revision-from-copy; `drawing_set_title_block` stamps project/drawing_no/revision/sheet/
  scale/approvers on the sheet.
- **Testing on the live estate:** experiment only on `hilir-test*` drawing names and
  `delete_output`/`drawing_delete` them afterwards. Never touch production drawings
  (e.g. `gas-chill-pfd`).

## Type matrix

| type | element kinds | export |
|---|---|---|
| `pfd` | tank, pump, heat_exchanger, column, compressor, vessel_*, air_cooler, fired_heater, reactor, mixer, splitter, filter, flare, ejector, instrument, battery_limit — **no valves** (P&ID detail) | DEXPI XML+JSON, SVG/PNG/PDF |
| `pid` | the PFD set **plus** 14 valves (gate/globe/check/ball/control/relief/…) and typed instrument bubbles | DEXPI XML+JSON, SVG/PNG/PDF |
| `plan` / `ga` | wall {x,y,len,dir:H\|V,thickness}, door {x,y,width,side}, column {x,y,size}, equipment {x,y,w,d,tag}, label | DXF, SVG/PNG/PDF |
| `plot` | boundary (closed polygon), road (centreline+width), equipment, pad, tie_in, north, scale_bar, label — metres | DXF, SVG/PNG/PDF |
| `iso` | **runs only** (`run_add`): direction-list sketches | DXF, SVG/PNG/PDF |
| `sld` | bus {len,orient,voltage}, transformer {kva}, breaker, switch, motor {kw}, generator, source, load, label + cable lines | DXF, SVG/PNG/PDF |
| — | parametric solids via `solid_create`: box, cylinder, pipe_spool, flange, plate → STEP+STL+metrics | STEP/STL |

Key recipes live in the agent guide §5; the field lists above plus `symbol_library` cover
the rest. Two traps worth naming: on a flange, `bolt_circle` is a **radius** while
`bolt_circle_dia` is the PCD — give one, not both. Doors do **not** cut walls (Phase B):
draw the wall in two segments and place the door in the gap.

## Material take-off (MTO)

`mto_extract(name)` is computed **server-side from the validated document** — never
recompute it yourself; it cannot drift from what was validated.

- pfd/pid → line list (tag/size/spec/from/to) + valve, instrument, equipment counts;
  `pipe_length_m: null` — a schematic has no lengths.
- iso → pipe length per run + auto-counted elbows (one per direction change) + the
  `fittings`/`valves` you declare on the run (at `run_add` time or later via `run_update`).
- plan/ga/plot → wall centreline metres, door/column schedules, equipment footprints,
  site/road areas. sld → equipment counts + cable route length.
- `fmt:"csv"` writes `<name>-mto.csv` (downloaded via `get_output`).

Honest boundary: fabrication-grade MTO (gaskets/bolts by flange spec, weld counts) still
needs commercial ISOGEN — this gives the line list, lengths, elbows and declared items.

## Retrieval & work-dir hygiene

Every writer returns the artifact **content** in its result while the **whole reply stays
under 64 KiB** (with `sha256`). Above that, the entry is metadata only — `filename`,
`bytes`, `mime`, `sha256`, `content_omitted` — plus a `hint`; fetch it with
`get_output(filename)` in 64 KiB chunks (bare filenames only, never a path; hard read
ceiling 64 MiB). **STL is always returned by reference** by default — a mesh is large and
rarely wanted inside a reply; the STEP still inlines when it fits the budget.

`list_outputs()` inventories deliverables with a namespaced owner (`drawing:<name>` /
`solid:<name>`) and an `orphan` flag. Attribution is the manifest entry, or — for files
that predate owner tracking — a stem matching a live drawing (`<name>.*`,
`<name>-mto.csv`); those are recorded on the next `list_outputs` and are **not** orphans.
`delete_output` removes one; **`prune_outputs(dry_run=True)` first, then
`dry_run=False`** to clear leftovers — a live drawing's files are never swept while the
drawing exists. `drawing_delete` removes the drawing's own outputs by default.

## Failure → action (top cases)

| You see | Do |
|---|---|
| `has no nozzle 'x'` | `symbol_library(type)`; use a listed nozzle key |
| `cannot render/export invalid drawing` | `drawing_validate`, fix the listed problems |
| valve used as line endpoint / dangling ref | valves go inline via `"through"`; endpoints must be equipment nozzles or points |
| `export format 'x' not supported for <type>` | pid/pfd → `dexpi`; plan/ga/plot/iso/sld → `dxf`; any → `svg`/`png`/`pdf` |
| `element 'x' is referenced by lines …` | `line_delete` those lines first |
| `bolt circle too small/too large` | `bolt_circle` is a radius; use `bolt_circle_dia` for PCD |
| `filename must be a bare file name` | `get_output`/`delete_output` take a filename, never a path |
| 401 / 403 | re-authenticate / fix the `cad:generate` grant |

## Limits — state these honestly to the user

- **No auto-layout**: you choose every coordinate; overlaps are not detected.
- No wall/door boolean cuts, no dimension chains, no P&ID label collision avoidance.
- Isometrics are sketch-grade (direction-list projection, not fabrication isos).
- DEXPI export is structural (validated); property sets/classes are Phase B.
- Title block is a stamped block, not a full ISO 7200 frame (Phase B).
- **SLD draws and counts only** — sizing, protection coordination, arc flash and
  short-circuit results belong to Thunderbolt.

## Chaining the estate

- **DWSIM** mass/energy results (stream sizes, equipment) → become PFD/P&ID tags and
  `solid_create` vessel shells.
- **Thunderbolt** — run the electrical studies first, then draw the SLD; or import a
  validated Thunderbolt network straight in via `sld_import_network` (nodes+edges).
- **hilir-mygis** site facts / measured boundary coordinates → `plot` type; note the
  boundary coordinates are *your input* — no connector supplies lot geometry.
- **hilir-resources** man-hours and cost norms take the MTO quantities from `mto_extract`
  as their quantities.
