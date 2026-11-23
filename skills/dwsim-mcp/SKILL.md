---
name: dwsim-mcp
description: Run chemical process simulation through the HILIR DWSIM MCP server (dwsim.hilir.app) — build and solve steady-state and dynamic flowsheets (streams, 44 unit operations, reactions, PID control), mass/energy balances, pump curves with NPSHr, pipe hydraulics, shell-and-tube heat-exchanger rating, distillation screening, scenario sweeps and dynamics/PID-tuning studies. Use it whenever the user asks to simulate, model, flash, distil, compress, pump, heat or size a chemical process; asks for flowsheet results such as flows, duties, temperatures, pressures, pump head/NPSH or pipe pressure drop; asks for a dynamic response, PID tuning or interlock study; or asks why a flowsheet does not solve. Do NOT use it for drawings — PFDs, P&IDs, plot plans and isometrics belong to JELAS CAD (DWSIM renders a topology PNG only) — or for electrical studies (Thunderbolt owns the SLD, load-flow and short circuit).
---

# DWSIM — process simulation over MCP

`dwsim.hilir.app` runs the DWSIM chemical process-simulation engine headless on the
**HILIR fork** (70 tools: 58 upstream + 12 HILIR — `tools/list` is the live count, trust
it over this file). You build a *flowsheet* — compounds, streams, unit operations wired
port to port — and the engine calculates. **The server asks for what it needs**: judge
every gap as ask (`dwsim_flowsheet_degrees_of_freedom` / `dwsim_flowsheet_check`) →
deliver (the settable property) → describe. Never report "DWSIM can't" without checking
the DOF catalogue first — most gaps are settable inputs, not missing features. Every
tool is self-describing: trust the tool description over memory.

Deep reference, public and current: **read https://dwsim.hilir.app/llms.txt before a
complex build** (one-screen capability brief). With repo access
(`github.com/tenunan-dev/hilir-dwsim`) the deeper contracts are `AGENTS.md` (usage
contract) and `reference/measured/unit-op-inputs.md` (measured per-unit input list).

## Connect

```
URL:      https://dwsim.hilir.app/mcp     (Streamable HTTP, POST only — GET/DELETE
                                           answer 405, which is normal)
Headers:  Authorization: Bearer <credential>   (or x-api-key: <credential>)
```

The credential is an **estate key** from access.hilir.app with scope **`dwsim:run`**
(create a key → service DWSIM → role Runner), or an Access JWT with
`aud = https://dwsim.hilir.app`. **There are no `sk-` keys.** In this workspace the tools
normally appear via the hilir-libraries plugin, prefixed
`mcp__plugin_hilir-libraries_hilir-dwsim__…`; direct MCP clients register them under
their own server name (Claude Code:
`claude mcp add --transport http dwsim https://dwsim.hilir.app/mcp --header "Authorization: Bearer <credential>"`).
Protocol quirks that bite raw-HTTP callers: **stateless** (no session id — every request
stands alone), one JSON-RPC message per request (batching returns `-32700`), the payload
is the JSON *string* in `result.content[0].text` (parse it, don't index it), and an HTTP
200 can carry a JSON-RPC `error` — read the error member, never the status alone. 401 =
no/invalid credential; 403 with `required: dwsim:run` = not an estate key (every estate
key carries the scope) — ask the human, don't retry.

## The core loop — always close it

`create → add ALL compounds → set package → streams + units + connect → DOF = 0 → check → solve → read results → close`

1. **Compounds before objects.** Add *every* compound the flowsheet will ever need
   (`dwsim_thermo_add_compounds`) before building any stream or unit. Compounds added
   later poison the built objects (`COMPOUND_DRIFT`) and the flowsheet is unrepairable —
   rebuild. Database names only: `dwsim_thermo_list_compounds` (pass `query`, else ~25 KB
   dumps).
2. **Property package, discovered.** `dwsim_thermo_list_property_packages` first; exact
   names, unique base names or short forms (`PR`, `SRK`, `IAPWS-IF97`).
3. **Drive the DOF to zero.** `dwsim_flowsheet_degrees_of_freedom` lists each unit's open
   slots with names and units; fill them (`dwsim_unitop_set`,
   `dwsim_pipe_profile_set`, geometry on an OrificePlate). Objects with holes come
   first. Never trust `set: true` alone — a value is only real when the solve uses it.
4. **`dwsim_flowsheet_check` before every solve.** Cheap. Catches missing specs, open
   reactor outlets and compound drift before you pay for the solve.
5. **Trust findings, not ok.** Since fork v7 `dwsim_solve_run` returns `ok:false` only
   when a **steady-state** object is unsolved or a blocker exists; controllers, gauges and
   other dynamics-only objects are never calculated in steady state and are listed under
   `not_applicable_steady_state` (each status entry carries `steady_state`).
   `dwsim_flowsheet_check` warns `CONTROLLER_UNWIRED` when a PID lacks a side of its loop.
   Read `findings[]` and fix what it names; `dwsim_solve_diagnostics` +
   `dwsim_explain_finding` when unclear.
6. **Read results from the fork-shaped payloads.** `dwsim_unitop_get_results` `summary`
   carries `duty_kW`, efficiency and every connected stream's calculated T/P/flows;
   per-unit keys stay in `properties`. `dwsim_stream_get_results` answers material *and*
   energy streams (energy returns `energy_flow_kW`); unflashed phase values come back
   `null`, not 0/NaN — that is "no data", not an error. An empty phase has no compounds
   and no numbers; a present phase carries the stream temperature and pressure.
   For the whole flowsheet in one call use `dwsim_flowsheet_stream_table` (fork v8):
   material rows (phase, T, P, flows, composition) plus energy-stream duty, values
   matching `dwsim_stream_get_results`.
7. **Save before an expensive solve** (`dwsim_flowsheet_save`), and close what you
   opened (`dwsim_flowsheet_close`).

## Working discipline (the rules that bite)

- **CalcMode travels with its value.** A spec set while `CalcMode` reads something else
  reports `UNIT_HAD_NO_EFFECT` and does nothing. Discover a unit's settable names by
  passing a deliberately wrong key to `dwsim_unitop_set` — the error lists them (long
  lists truncate at ~40 + "and N more"; repeat with another wrong key).
- **Cooler duty sign:** positive `DeltaQ` removes heat; `COOLER_HEATED` means flip it.
- **Values agree with their labels.** Input and output use the active unit system (SI
  default; `dwsim_flowsheet_set_units` switches SI/CGS). Fork v2 fixed the engine tables,
  so quote the label with the value — fugacity coefficient is dimensionless, Cp is
  kJ/[kg.K].
- **Connect atomically.** `dwsim_unitop_connect` applies every stream in the call as one
  set: a failure leaves nothing attached — fix the named stream/port and retry the same
  call.
- **Reactions.** `dwsim_reactionset_create` → `dwsim_reaction_add_conversion` /
  `_add_equilibrium` / `_add_kinetic` (compounds must already be in the flowsheet;
  kinetic orders are per stoichiometry compound, unlisted = 0). Reactors use the set via
  `ReactionSetID` (DefaultSet otherwise). `dwsim_reaction_edit` changes a reaction in
  place — identity, set membership and rank survive; stoichiometry is a full validated
  replacement. `dwsim_reaction_remove` drops it from every set.
- **A Gibbs reactor needs both outlets connected** (even when one phase is empty) and
  `dwsim_reactor_set_compounds` called once its feed is connected and specified —
  otherwise "Element Matrix is not defined". Loops need a **Recycle** block on a tearing
  stream.
- **PID.** Add the unit (`dwsim_unitop_add type=PIDController`) then *wire* it with
  `dwsim_dynamics_controller action=create` (pv/mv wiring; optional feedforward
  `ff_object`/`ff_property`). A create on an existing tag errors — use `action=set` to
  tune. `dwsim_dynamics_tune_pid` needs a stored state so trials are comparable.
  The controller list reports flowsheet **tags** with property descriptions
  (`controls` / `manipulates`, e.g. `"S2.PROP_MS_0 (Temperature)"`) and keeps the engine
  address in `controls_id` / `manipulates_id`; `dwsim_flowsheet_check` warns
  `CONTROLLER_UNWIRED` until both sides are wired — a steady-state solve never moves an
  unwired controller and reports `ok:true` either way.
- **Dynamics in order.** `dwsim_dynamics_inspect` → `_check` → `_setup` (schedule) →
  `_object` (pressure-flow spec, valve calc mode) → `_properties` (property ids are not
  guessable — call it before monitor/event/controller) → `_monitor` → `_event` →
  `_state` save → `_run` → `_status` / `_series` (series is a decimated preview;
  `_export` writes the full CSV) → `_analyze`. Events take property **ids** (`PROP_CL_3`),
  never display names; a display name may still register a broken event — check
  `_event action=list`, `clear` the set, re-add with the id.
- **Interlocks.** `dwsim_indicator_add` (Analog/Digital/Level gauges with LL/L/H/HH
  alarms) feeds `dwsim_dynamics_cem` (alarm fires → set a property on a target).
- **File discipline.** Scratch in `/tmp` (container-local, flat paths, no mkdir —
  nested paths fail); `/opt/dwsim-mcp/data` is the persistent, nightly-backed *shared*
  store — never write tests there. No tool deletes a file: what you save stays — say
  what you left behind. Flowsheet *objects* can be deleted (`dwsim_object_remove`,
  `dwsim_unitop_disconnect`, `dwsim_reaction_remove`) — no undo, verify with
  `dwsim_flowsheet_list_objects`.
- **Test on the live estate under `hilir-test*` names** and `/tmp` scratch paths only.
- **Output ceilings.** `dwsim_flowsheet_get_xml` truncates at `max_chars` (default
  50000) — use `dwsim_flowsheet_save` for the full file. `dwsim_graphic_screenshot`
  returns the PNG as a real image content block (the text block carries only
  `size_bytes`) — render it, or `dwsim_graphic_screenshot_to_file` for a file.
- **Filters fail open.** `dwsim_flowsheet_list_objects type_filter` accepts exactly
  `"streams"` / `"unitops"` — anything else silently returns everything. A `Mixer`
  reports `type:"NodeIn"` and a `Splitter` `"NodeOut"` — DWSIM's own naming, not a
  missing object.

## Sizing you can actually get

| Unit | What |
|---|---|
| `HeatExchanger` | Two passes, screening then rating: `CalculationMode:"CalcArea"` with a defensible U, then `"ShellandTube_Rating"` with real geometry → LMTD + correction factor F, MITA, tube/shell Reynolds & velocities, nozzle ρv², areas, weights. Hardware is settable (`PROP_HX_*`: shell/tube dia, length, count, pitch, passes, baffles, fouling). |
| `Pipe` | `dwsim_pipe_profile_set` (sections of length/elevation in m + inner diameters in mm; 0.045 mm carbon-steel roughness default) — then it solves like any unit; after the solve, per-section velocity, Reynolds, friction/static dP, holdup and flow regime. No `.dwxml` hand-authoring needed since fork v3. |
| `Pump` | `dwsim_pump_curve_set` (head vs flow required; NPSHr and efficiency optional) + `CalcMode:"Curves"` → interpolated head, **NPSH required** and efficiency at the operating flow (`summary.curve_operating_point`). Scalar modes (`Delta_P`/`OutletPressure`) still work for a quick duty point. `Delivered Volumetric Flow` (`PROP_PU_12`) is the liquid volume the pump passes, in every mode since fork v7.1. A `Temperature Difference` carries the engine's delta-unit string (`C.` with the trailing dot) — it is a *difference*, so do not convert it as an absolute temperature. |
| `DistillationColumn` | Screening: estimated diameter/height, column pressure drop, flooding alarm, stage efficiency. |
| `Valve` | Kv(max)/actual Cv from size and opening, cavitation alarm. `OrificePlate` solves with settable geometry (the DOF table reports `supported:false` — the geometry is still settable). |

## Failure → action (top cases)

| You see | Do |
|---|---|
| 401 + challenge | no/invalid credential — the human supplies an estate key or Access JWT (there are no `sk-` keys) |
| 403 `required: dwsim:run` | not an estate key (every estate key carries it) — ask the human to check the key; don't retry |
| `-32602 Missing required parameter: X` | read the tool's `inputSchema`, supply `X` (compounds take `names`, a package takes `name`) |
| `-32700` parse / deserialize array | one JSON-RPC object per request — no batching |
| 405 | you sent GET/DELETE — POST only |
| `ok:false`, blockers in `findings[]` | fix what the findings name; `_diagnose` → `explain_finding` for detail |
| `UNIT_HAD_NO_EFFECT` | the unit's CalcMode doesn't read your spec — set both together, re-solve |
| `COOLER_HEATED` | duty sign flipped — positive `DeltaQ` removes heat |
| `REACTOR_OUTLET_OPEN` | connect every material outlet; a Gibbs needs both, even when one phase is empty |
| `SPEC_MISSING` | fill the named specification or switch the calculation mode |
| `COMPOUND_DRIFT`, or every later solve `KeyNotFoundException: '<compound>'` | compounds were added after objects were built — unrepairable; rebuild, all compounds first |
| `KeyNotFoundException` from `add_compounds` | database names only — `thermo_list_compounds` with `query`; there is no custom compound |
| PID tuning refuses ("doesn't have a PID controller") | `unitop_add type=PIDController` + `dynamics_controller action=create` first; then tune with `action=set` |
| `IndexOutOfRangeException` from `dynamics_event` | a display property name was passed (it may still have registered — `_event action:list`, `clear`, re-add with the id, e.g. `PROP_CL_3`) |
| phase values `null` | never flashed — "no data", not an error |
| `get_xml` truncated at 50000 chars | intended — raise `max_chars` or use `flowsheet_save` |
| `-32602` naming `flowsheet_id`, or "doesn't exist" after a restart | create or `dwsim_flowsheet_load` one — state lives in the engine process |

## Limits — state these honestly to the user

- **No drawings.** The screenshot is a topology PNG — no PFD, no P&ID, no instrument
  tags, line numbers or control loops. Those drawings come from JELAS CAD.
- **No new chemistry.** 1621 compounds, database lookup only; no custom/hypothetical
  compounds, and no external unit operations are installed.
- **Not fabrication design.** TEMA/ASME mechanical design, vendor tray/packing
  hydraulics and pipe stress are outside DWSIM — present ratings as screening/rating
  pending code calc + vendor confirmation.
- **State is process-bound.** `flowsheet_id` lives until the engine restarts; anything
  that must survive goes through `dwsim_flowsheet_save` under `/opt/dwsim-mcp/data`.

## Chaining the estate

- **JELAS CAD** takes the results (stream flows, sizes, duties) as PFD/P&ID tags and
  `solid_create` vessel shells; DWSIM has no drawing surface of its own.
- **Thunderbolt** owns the electrical side (SLD, load-flow, short circuit, arc flash) —
  no overlap with DWSIM.
- **hilir-resources** estimates take quantities via the CAD MTO, which should already
  reflect DWSIM-derived sizes.
