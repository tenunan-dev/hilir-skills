---
name: hilir-cad
description: Generate engineering drawings and 3D solids through the JELAS CAD MCP server (cad.hilir.app) — PFDs, P&IDs, site plot plans, building plans/GAs, piping isometrics, electrical single-line diagrams and parametric solids, with deterministic material take-offs delivered as issued A3/A1 sheets (SVG/PNG/PDF), DXF, DEXPI or STEP/STL. Use it whenever the user asks to draw, sketch, generate, revise or export a process flow diagram, P&ID, plot plan, layout, general arrangement, isometric or SLD; asks for a material take-off (MTO), line list, equipment/door/column schedule or an H&MB stream table; asks to issue several sheets as one set, or to compare two revisions; asks for DXF, DEXPI, STEP or STL output; or wants a parametric 3D part. Do NOT use it for electrical analysis (Thunderbolt owns load-flow/short-circuit) or flowsheet simulation (DWSIM) — this service draws and counts, it does not solve those.
---

# JELAS CAD — drawing generation over MCP

`cad.hilir.app` is a headless drafting service: you edit a **validated drawing document**;
the engines (pyDEXPI, ezdxf, build123d) produce the deliverables, and every render/export
composes a real drawing sheet. You never draw pixels — you place elements and connect them.
The authoritative deep reference is the server's own guide — **read
https://cad.hilir.app/agent-guide before a complex drawing**; this skill is the working
summary. Every tool is self-describing: trust the tool description over memory.

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
same loop (symbol_library first, valve-`"through"` discipline, type matrix, sheet/set
rules, MTO, outputs hygiene, honest limits); read it before your first tool call. This
skill and `GET /agent-guide` carry the deep detail on top of it.

## The core loop — always close it

`symbol_library(type) → drawing_create → element_add / line_add (run_add for iso) → drawing_validate → drawing_render / drawing_view → drawing_export`

1. **`symbol_library(type)` first, every type.** Allowed kinds, required fields, nozzle
   keys and the line fields are not guessable — `tank` N1/N2, `pump` S/D, `heat_exchanger`
   SI/SO/TI/TO, `column` TOP/FEED/BOTTOM, valves I/O, instruments none. The response also
   states `element_fields` (`attrs`, `package_boundary` w/d), `line_fields`
   (`stream_no`, `circuit`, `kind:"signal"`) and the sheet rule.
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
- **An instrument bubble prints its own tag.** `"PIC 1101"` renders `PIC` over `1101`
  *inside* the circle — never add a separate label element for an instrument.
- **Annotate, don't fake.** `off_page` names its partner sheet (`partner`), `tie_in`
  marks an existing connection, `circuit` demarcates a design circuit (its `letter`),
  `sample_point` carries the analyser service; a line's `stream_no` prints as `[S1]`.
- **Always supply `size` and `spec` on lines/runs** — they feed the MTO line-list columns,
  not just the sheet.
- **Errors carry the problem list** (`isError: true`). Fix the document, re-call. Never
  work around a rejection.
- **Revision workflow:** `drawing_open` → `element_update`/`line_add`/`run_update` →
  `drawing_validate` → `drawing_render` + `drawing_export`. `drawing_duplicate` for a
  revision-from-copy, `drawing_rename` to re-key a document (its deliverables follow),
  and `drawing_diff(name_a, name_b)` to write the revision note from evidence.
- **Read narrowly when the document is big:** `element_list(name, kind?)`, `line_get`,
  and `drawing_open` only when you need the whole document.
- **Testing on the live estate:** experiment only on `hilir-test*` drawing names and
  `delete_output`/`drawing_delete` them afterwards. Never touch production drawings
  (e.g. `gas-chill-pfd`).

## Type matrix

| type | element kinds | export |
|---|---|---|
| `pfd` | the process set (tank, pump, heat_exchanger, column, compressor, vessel_vertical/horizontal, air_cooler, fired_heater, reactor, mixer, splitter, filter, flare, ejector, instrument, battery_limit), the **power/utility set** (gas_engine {FUEL,AIR,EXHAUST,COOL_IN,COOL_OUT,POWER}, generator, exhaust_stack, scr_catalyst_bed, dry_radiator, basin, oil_water_separator, package_boundary {w,d}), the **detailed-PFD valves** (control, gate, ball, check, relief) and the **annotations** (off_page, tie_in, circuit, sample_point) — 34 kinds | DEXPI XML+JSON, SVG/PNG/PDF |
| `pid` | the PFD set **plus** the remaining valve detail (globe, butterfly, plug, needle, orifice_plate, blind, reducer, strainer, rupture_disc) and the typed bubbles (transmitter, controller, indicator, analyzer) — 47 kinds | DEXPI XML+JSON, SVG/PNG/PDF |
| `plan` / `ga` | wall {x,y,len,dir:H\|V,thickness}, door {x,y,width,side}, column {x,y,size}, equipment {x,y,w,d,tag}, label | DXF, SVG/PNG/PDF |
| `plot` | boundary (closed polygon), road (centreline+width), equipment, pad, tie_in, north, scale_bar, label — metres | DXF, SVG/PNG/PDF |
| `iso` | **runs only** (`run_add`): direction-list sketches | DXF, SVG/PNG/PDF |
| `sld` | bus {len,orient,voltage}, transformer {kva}, breaker, switch, motor {kw}, generator, source, load, label + cable lines | DXF, SVG/PNG/PDF |
| — | parametric solids via `solid_create`: box, cylinder, pipe_spool, flange, plate → STEP+STL+metrics | STEP/STL |

A **detailed PFD** is the normal deliverable: process equipment *and* the control,
on/off, check and relief valves a licensor package shows. The split that remains is
instrumentation detail — the extra valve styles and typed bubbles are `pid` only.

Key recipes live in the agent guide §5; the field lists above plus `symbol_library` cover
the rest. Two traps worth naming: on a flange, `bolt_circle` is a **radius** while
`bolt_circle_dia` is the PCD — give one, not both. Doors do **not** cut walls (Phase B):
draw the wall in two segments and place the door in the gap.

## The sheet

Every render and export composes a real sheet — the document is the content, the frame is
presentation, so DXF and DEXPI keep model coordinates while SVG/PNG/PDF get the sheet.

- **Paper:** **A3 landscape by default**; A1 with
  `drawing_set_title_block {"sheet_size": "A1"}` (any other value is refused). Content is
  scaled to fit the drawing area, centred, and clipped there, so nothing paints over the
  title block.
- **Frame:** double border (20 mm binding edge, 10 mm elsewhere), zone grid (one zone per
  ≤50 mm — 8×6 letters/numbers on A3), ISO-7200-style title block (180×55 mm) with
  project, drawing no, revision, sheet, scale, drawn/checked/approved, date, client.
- **Revision table:** `title_block.revisions`, a list of `{rev, date, description, by}`.
- **Notes:** `title_block.notes` (wrapped into the notes area).
- **Stream table:** see below — it prints one row per supplied stream in the table band.
- A PDF export lands on the true paper size (A3 = 1190.55 × 841.89 pt).

Deliver the sheet, not a bounding box: `drawing_view` for the user to see it,
`drawing_export(fmt="pdf")` for the issue copy.

## Material take-off (MTO)

`mto_extract(name)` is computed **server-side from the validated document** — never
recompute it yourself; it cannot drift from what was validated.

- pfd/pid → line list (tag/size/spec/from/to), valve counts by kind, `equipment`, and the
  instrument registers: `instruments` (`function`/`loop` parsed from the tag, plus any
  `attrs`), `analysers` (`measured_property`, `service`), `flowmeters` (`purpose`, `type`,
  `accuracy`, `criticality`), `design_circuits` (lines carrying a `circuit`).
  `pipe_length_m: null` — a schematic has no lengths. Buckets follow the symbol registry,
  not the kind spelling: typed bubbles (transmitter/controller/indicator/analyzer) are
  `instruments`, and blind, strainer, orifice_plate, rupture_disc and reducer are `valves`.
- iso → pipe length per run + auto-counted elbows (one per direction change) + the
  `fittings`/`valves` you declare on the run (at `run_add` time or later via `run_update`).
- plan/ga/plot → wall centreline metres, door/column schedules, equipment footprints,
  site/road areas. sld → equipment counts + cable route length.
- `fmt:"csv"` writes `<name>-mto.csv` (downloaded via `get_output`); it carries one section
  per register above.

Honest boundary: fabrication-grade MTO (gaskets/bolts by flange spec, weld counts) still
needs commercial ISOGEN — this gives the line list, lengths, elbows and declared items.

## Sets, stream register, revision diff

- **H&MB stream register.** `drawing_set_stream_table(name, streams_json)` attaches the
  register the sheet prints — **supplied** data, not solved by this service: one row per
  stream, columns `id`, `name`, `phase`, `T`, `P`, `kg/h`, `kmol/h` then every
  `composition` key in register order. Values are scalars; composition is the only object.
  Duplicate ids and nested values are refused. Pass `[]` to clear it.
- **Sheet sets.** `sheet_set_create(name, sheets_json, title?, namespace?)` orders member
  drawings as an issue set, stamps `sheet n/N` into each member's title block, and builds
  the index + legend (from the symbol categories actually used).
  `sheet_set_validate(name)` checks each sheet by its own engine, tag uniqueness inside the
  set namespace (a different namespace may reuse tags), reciprocal `off_page` connectors
  and unique `stream_no` across the set — and names the offending sheet. A partner may be
  the drawing name, `2`, `2/5` or `Sheet 2`.
- **Revision note.** `drawing_diff(name_a, name_b)` returns added, removed and changed
  elements/lines/runs with the field values on both sides, plus document-level changes
  (title, title block, stream register) — the evidence for a revision note.

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
| `kind 'x' not allowed on <type>` | it is detail for another type — `pid` for the extra valve styles and typed bubbles |
| `cannot render/export invalid drawing` | `drawing_validate`, fix the listed problems |
| valve used as line endpoint / dangling ref | valves go inline via `"through"`; endpoints must be equipment nozzles or points |
| `export format 'x' not supported for <type>` | pid/pfd → `dexpi`; plan/ga/plot/iso/sld → `dxf`; any → `svg`/`png`/`pdf` |
| `element 'x' is referenced by lines …` | `line_delete` those lines first |
| `sheet_size must be one of ['A1', 'A3']` | fix the title-block field, not the render call |
| `off-page … has no reciprocal connector` | add the matching `off_page` on the partner sheet |
| `tag 'X' appears on sheet 1 … and also on sheet 2` | rename one; tags are unique inside the set namespace |
| `bolt circle too small/too large` | `bolt_circle` is a radius; use `bolt_circle_dia` for PCD |
| `filename must be a bare file name` | `get_output`/`delete_output` take a filename, never a path |
| 401 / 403 | re-authenticate / fix the `cad:generate` grant |

## Limits — state these honestly to the user

- **No auto-layout**: you choose every coordinate; overlaps are not detected.
- No wall/door boolean cuts, no dimension chains, no P&ID label collision avoidance.
- Isometrics are sketch-grade (direction-list projection, not fabrication isos).
- DEXPI export is structural (validated); property sets/classes are Phase B.
- The sheet is an A3/A1 frame with title block, revision table, notes and the stream
  table — but there is **no BOM ballooning, no revision cloud and no automatic scale
  notation**; write "NTS" or the scale you used.
- The stream table and the MTO registers are **supplied/derived data**; this service does
  not solve mass or energy balances — DWSIM does.
- **SLD draws and counts only** — sizing, protection coordination, arc flash and
  short-circuit results belong to Thunderbolt.

## Chaining the estate

- **DWSIM** mass/energy results (stream sizes, equipment) → become PFD/P&ID tags, the
  `drawing_set_stream_table` register, and `solid_create` vessel shells.
- **Thunderbolt** — run the electrical studies first, then draw the SLD; or import a
  validated Thunderbolt network straight in via `sld_import_network` (nodes+edges).
- **hilir-mygis** site facts / measured boundary coordinates → `plot` type; note the
  boundary coordinates are *your input* — no connector supplies lot geometry.
- **hilir-resources** man-hours and cost norms take the MTO quantities from `mto_extract`
  as their quantities.
