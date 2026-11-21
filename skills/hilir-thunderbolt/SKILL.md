---
name: hilir-thunderbolt
description: Electrical power-systems calculation engine over the hilir-thunderbolt MCP server — a single-line-diagram data model with AC load flow, IEC 60909-0 short circuit, cable ampacity and voltage-drop checks, IEC 60255 protection coordination, static motor starting, N-1 contingency, IEEE 1584-2018 arc flash, SLD export, and estate project storage. Use when a task involves buses, transformers, cables, fault levels, voltage drop, breaker settings, incident energy, or single-line diagrams — and state the study case (3ph/2ph, max/min) with every fault figure.
---

# hilir-thunderbolt — electrical studies over a network document

One MCP server, 31 tools (27 engine + 4 Worker-served project tools). The server
computes from a network document you author as data; nothing is inferred from a
picture you did not specify. Call `get_capabilities` first — it lists what is
implemented, what is deliberately NOT, the refusal conditions, and the unit of
every field.

## 1. Authoring loop (stateless)

Authoring tools (`create_network`, `add_bus`, `add_source`, `add_transformer`,
`add_cable`, `add_load`, `add_motor`, `add_breaker`, `update_element`,
`remove_element`, `set_assumptions`) are **stateless**: each takes the whole
`network` document and returns the modified one. **Thread the returned document
into the next call** — the reply is a wrapper `{ok, network, assumptions}`, so
pass `network` onward or the solver refuses with
`invalid_network: Extra inputs are not permitted`. The engine stores nothing
itself.

- Start from `get_example_network` (a known-valid shape — `radial_lv`,
  `mv_ring`, `gen_utility`; `list_example_networks` for the catalogue) or
  `create_network`, then change the values. Buses first — everything attaches
  by bus id.
- Batch edits: `apply_changes` threads the document internally in one round-trip.
- A field you cannot name: `get_network_schema` gives the field-level schema.
- Ratings are nameplate values, not study results. Nameplate vs bus voltage
  mismatches are reported as notes (the result is then scaled by the tap
  ratio) — relay those notes, they change the answer.

## 2. Before any study

`validate_network` first: zero solver cost, catches dangling references,
duplicates, unreachable nodes. Its `notes` are not errors — they are warnings
that change the answer and must reach the user. A refusal (`ok:false`) means the
document cannot be solved at all; fix it before paying for a study call.

## 3. Studies and their gotchas

| Tool | Returns | Non-negotiable care |
|---|---|---|
| `run_loadflow` | bus voltages, branch loading, losses, violations | Report nothing if `converged:false`. Loading is measured against **derated** ampacity — a cable with no `max_i_ka` is reported `n/a`, never invented. |
| `run_shortcircuit` | IEC 60909 I_k″, i_p, I_th, S_k″ per bus | State **fault type (3ph/2ph/1ph) and case (max/min) with every figure**. Use max to size switchgear, min to check protection will still operate. |
| `run_cable_check` | derated ampacity utilisation + voltage drop | A check, not a catalogue selector — choosing a section needs the manufacturer's tables. `null` in a check means "data absent", never "passed". |
| `run_coordination` | series-pair selectivity on IEC 60255 curves | The 0.2 s margin is a **design default, not a standard**. Devices that cannot be evaluated are listed in `unchecked` with the reason — never silently dropped. |
| `run_motor_start` | static locked-rotor voltage dip | **Not a dynamic starting study** — no acceleration, inertia, or time-to-speed. Say so when quoting the dip; the 15 % limit is a usual DOL design target, not a standards claim. |
| `run_contingency` | N-1 over cables, transformers, optionally sources | Single outages only — N-2 is not implemented. |
| `run_arc_flash` | IEEE 1584-2018 incident energy + AFB | Needs `arc_flash_*` geometry on each breaker — without it the breaker is listed in `skipped`, not computed. Refused for DC and >15 kV. |

### Short-circuit specifics

- **1ph needs zero-sequence data on every element on the fault path** (`x0x_ratio`/
  `r0x0_ratio` on the utility, `r0/x0/c0` on cables, `vk0/mag0/si0` on transformers).
  It is refused, never approximated, until present — a Dyn11 blocks zero sequence
  between LV and HV.
- `case='min'` needs `ssc_min_mva` on every utility source; a generator-only
  network cannot do short circuit (a utility infeed with `ssc_mva` is required);
  a generator needs `rdss_ohm`. All refused rather than guessed.
- For `1ph` only `ikss_ka` is produced — quote I_k1 alone; i_p, I_th, S_k″ and
  κ are null with a note.
- kA values are **not comparable across transformers** — each bus's figure comes
  from its own voltage factor and topology. Never sum or average fault currents.

### Arc flash specifics

- Per breaker: bolted fault from the 3ph/max study, arcing current (worst of the
  full and reduced cases), clearing time from the breaker's own IEC 60255 curve
  (or an explicit `arc_flash_clearing_time_s`), incident energy, AFB, and
  `min_arc_rating_cal_cm2`.
- **PPE selection is NFPA 70E Art. 130's call** — the table method is a different
  method, not interchangeable with the incident-energy result. Report the
  incident energy and the PPE arc rating it implies; do not label a result with
  a PPE category.
- Do not check engine figures against a remembered standard worked example —
  reconstructing IEEE 1584 annexes from memory produces plausible wrong numbers.

## 4. Assumptions — echo them with every number

`set_assumptions` changes what every study assumes; each study result echoes the
assumptions actually used. Quote the echo when handing figures over. Two need
the user's word, never a silent default:

- `lv_tolerance_percent` (6 or 10) sets the IEC 60909 c factor for LV buses —
  **ask which band the installation is designed to**.
- `motor_contribution` — excluding motors from I_k″ is a legitimate conservative
  choice, but it must be **stated** whenever the result is quoted.
- `tk_s` (clearing time for I_th), `conductor_endtemp_c` (70 °C for PVC, 90 °C
  conservative default for XLPE), and the ambient/soil temperatures derate
  ampacity and the minimum case — they are not decorative.

## 5. Persistence (Worker-served, estate storage)

The four project tools are answered by the Worker from estate D1, not the
engine; ownership is the caller's own subject:

- `project_list` — my projects (id, name, latest revision).
- `project_save` — without `project_id` creates revision 1; with `project_id`
  appends a new revision (**append-only**, number minted server-side; `note`
  ≤ 200 chars).
- `project_open` — newest revision, full study-ready network document.
- `project_archive` — soft delete: hidden from `project_list`, data survives.

Use these to keep a document across conversations or to version a study
(append a revision per design case rather than overwriting).

## 6. Export

- `export_diagram` — JSON geometry (buses, branches, devices, bbox) for an SLD
  viewer or a CAD service.
- `export_svg` — an SVG picture; write the `svg` string to a file to open it.
- Positions (`x`/`y`) are presentation-only canvas coordinates. Unplaced
  elements render on a deterministic auto-grid and are listed in `at_origin` —
  that is a layout gap, not an engineering one. **DXF is not produced**; for a
  real single-line drawing use the CAD library, not this one.

## 7. Rules that keep answers defensible

- Never report numbers from a non-converged solve; the result says why it did
  not converge.
- Never approximate a refused study. If the engine refuses (zero-sequence
  missing, min case without `ssc_min_mva`, generator without `rdss_ohm`), fetch
  the data from the user or the datasheet instead of estimating it.
- The engine will not invent ampacity (`max_i_ka` must come from the datasheet)
  and does not reproduce IEC current tables — do not read cable selection into
  a cable-check result.
- Not implemented (do not read into output): harmonics/transients/EMT, N-2,
  dynamic motor starting, fuse melting curves, DC arc flash, DXF. If a task
  needs one of these, say so instead of stretching a study.
- Out-of-scope on purpose: the engine does not size equipment for you — it
  checks what you declare. Combine with `hilir-techlib.find_figures` for
  equipment ratings and cost basis, and `hilir-cad` for drawing deliverables.
