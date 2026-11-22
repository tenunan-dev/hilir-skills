---
name: mygis-mcp
description: Use the mygis Malaysia GIS MCP server — 32 read-only tools (worldwide forward/reverse geocoding and place detection, Malaysia admin/cadastral lookup, land cover and land use, forest and Sarawak overlays, terrain, bathymetry, routing, static/satellite maps with Wayback history, 26 self-hosted snapshot layers including Canada/USGS oil & gas, construction site lens and the site_report aggregate) over free/open data. Covers the stateless connection (plain POST, no session id), authentication (Access JWT or estate key with catalog:read), the grouped tool loop, measurement and map recipes, and the honesty rules (unstated blanks, coverage ≠ absence, centroid distances, JPPH = reference not transactions). Use when asked to query mygis, mygis.hilir.app or the hilir mygis plugin, or for any Malaysia location question — a point's admin chain, what's on the land there, zoning/flood/slope for a project site, plot comparison, ports/quarries near a site, or a map screenshot with numbers instead of pixel estimates. Geocoding/reverse geocoding work worldwide; overseas snapshot layers (Canada, Suriname, global oil & gas) are catalogue/query-only and are not part of the Malaysia admin/land tools.
---

# mygis MCP (Malaysia GIS)

mygis is a **read-only** GIS toolkit for Malaysia over free/open data (OSM, JUPEM,
PLANMalaysia, Forestry FMRS, DOE, Esri, GEBCO, JPPH, Statistics Canada, AAFC, USGS,
GEM, BCER, AER) plus **26 self-hosted snapshot layers** and the NRCan land-cover
raster overlay. **Every tool requires the `catalog:read` scope; nothing writes.**
Coordinates are decimal degrees `[lat, lon]` and every result carries attribution
and `retrieved_utc` — keep the attribution in anything you show a user.

## Connection

| | |
|---|---|
| MCP endpoint | `POST https://mygis.hilir.app/mcp` |
| Web map | `https://mygis.hilir.app/` (same app, same credential) |
| Auth header | `Authorization: Bearer <credential>` — `x-api-key: <credential>` is accepted identically |
| Credential | A hilir Access JWT with `aud = https://mygis.hilir.app` (the resource origin, **never** the client name `mygis`), or a person's estate key with `catalog:read`. Obtain/refresh at [access.hilir.app](https://access.hilir.app) |
| Accept header | `application/json, text/event-stream` (responses may arrive as SSE `data:` lines) |
| Rate limit | Per-key 60/min — exceeding returns `429` with `Retry-After`; respect it |

**Stateless Streamable HTTP — no session handshake.** Every POST is standalone: no
`mcp-session-id`, no `initialize`/`notifications/initialized`. Send the method straight:

```bash
TOKEN="…"   # Access JWT (aud https://mygis.hilir.app) or estate key with catalog:read
H=(-H "Authorization: Bearer $TOKEN" -H "accept: application/json, text/event-stream" \
   -H "content-type: application/json")

curl -fsS "${H[@]}" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' https://mygis.hilir.app/mcp
curl -fsS "${H[@]}" -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"place_detect","arguments":{"lat":2.7358,"lon":101.6686}}}' https://mygis.hilir.app/mcp
```

MCP client config (Claude Code, VS Code, Cursor, Copilot agents — same shape everywhere):

```json
{ "mcpServers": { "mygis": {
    "type": "http",
    "url": "https://mygis.hilir.app/mcp",
    "headers": { "Authorization": "Bearer <Access token or estate key>" }
} } }
```

Credential rules that were deliberately settled and must not be "simplified":

- **There are no `sk-` keys.** They were deleted estate-wide; never reintroduce or invent them.
- A 401 means missing/expired/wrong-audience; a 403 means the credential is valid but lacks
  `catalog:read` (`{"error":"forbidden"}` shape). Never widen a scope to make a call work —
  refresh the credential instead.
- **Don't hammer a bad key.** Key-shaped bearers are introspected at Access, which caps its
  endpoint at 50 req/10 s per edge+IP; mygis opens a local 10 s circuit on repeated
  401/403/429. If a 401 persists, wait it out and mint a fresh key at Access — retrying the
  same key rapidly blocks your IP longer.
- Scripted probes against hilir hosts must send a real User-Agent — Cloudflare answers
  library-default `python-urllib` fingerprints with error 1010 (curl is fine).

## The 32 tools, by what you're actually doing

**LOCATE** — `geocode` name/address → coordinates **worldwide** (MyGeoName first for Malaysia,
Nominatim for the rest; optional `country_code`); `reverse_geocode` coordinates → address
worldwide; `place_detect` one call: place name + address + geocode round-trip drift, plus
Malaysia-only admin chain + JUPEM UPI hierarchy when the point is in Malaysia — the
"where is this coordinate" answer; `admin_lookup` state/daerah/Parlimen (P.xxx)/DUN (N.xxx)
at a point, offline and instant
(federal territories have **no DUN** — absence is by design); `cadastral_lookup` JUPEM UPI
negeri/daerah/mukim/seksyen with codes (admin hierarchy, **NOT** lot parcels; seksyen absent
outside gazetted cities); `mukim_lookup` FMRS mukim + forest district.

**WHAT IS THERE** — `whats_here` every OSM feature at the point (≤40 items); `land_cover`
Sentinel-2 10 m class (Trees/Built/Crops/Water/…) with auto-retry of the previous year on
no-data; `land_cover_stats` % + ha mix over a bbox or radius (grid-sampled estimate, not a
census); `land_use` i-Plan planning category (~2017 epoch, **Peninsular only**, dev-server
backed — transient timeouts error cleanly, retry); `doe_landuse` DOE 2018 (**Johor only,
partial**); `doe_klang` factories/chemical inventory (CAS numbers)/public complaints
(**Klang study area only** — empty elsewhere means not covered, not nothing there);
`forest_status` HSK reserve name, forest district, gazettement date/number/ha, compartment,
function class (Peninsular; a **stopped FMRS service answers "unknown", not "not in a
reserve"**); `sarawak_land_status` Sarawak overlays (PFE, logging/oil-palm concessions, LPF,
RSPO, protected areas, peatland); `features_in_area` raw simplified GeoJSON for a
bbox/polygon/radius (≤5000 features, bbox ≤0.25 deg², OSM ways only — no multipolygon
expansion).

**PROJECT SITE LENS — start with `construction_site`**: structure-plan zoning (density,
height limit, plinth %, gazette), the cadastral lot, flood zone with its ARI return period,
slope band + hazard class, brownfield, telco/drains nearby, elevation + contour band,
imagery date. Blank source fields come back `null` and are named in `unstated`;
`not_available` lists what mygis cannot answer (soil/geology, groundwater, utility capacity,
permits, cost) — quote those honestly instead of guessing. Then `road_access` for the road
that serves the site (OSM class/name/ref/surface, distance to nearest **mapped vertex** — a
lower bound, not a survey); `nearest` POIs ranked by road distance/drive time (35 category
presets, ≤20 results); `poi_search` raw POI search (35 presets or OSM tags, ≤50 results,
`include_road_distance:true` adds drive data per result); `jpph_comparables` valuation-
**reference** buildings within a radius — reference records, never transactions.

**SNAPSHOT LAYERS (self-hosted, no upstream call)** — `layers_list` is the catalogue first:
id, group, **scope**, country, feature count, fields, bbox, source, licence, **access**,
extraction date; `layer_query` fetches features near a point nearest-first. `scope`
separates the Malaysia thematic layers (`malaysia`) from the **Overseas** panel
(`overseas`: Canada — StatCan boundaries, AAFC ecumene/soils, BCER/AER oil & gas;
Suriname/regional/global — USGS + GEM). Coverage is per layer: Malaysia zoning/brownfield/
lots are Melaka extracts, contours Johor, flood/slope Peninsular-wide, telco/drains
partial; BCER is British Columbia only; AER is Alberta and `access: "internal"`
(non-commercial, do not republish). Canada land cover is a raster map overlay
(`canada_landcover`), not a `layer_query` snapshot — **check a layer's bbox before
treating an empty result as a finding**. On contour layers each hit carries
`elevation_m` — the surveyed 5 m band, not a spot height — plus a contour summary.

**TERRAIN & WATER** — `elevation` DEM at a point (±10 m) with optional profile along
waypoints (gain/loss/min/max); `seabed_depth` GEBCO value (negative = metres below sea
level; ≈450 m cells, not for navigation).

**MAPS** — `static_map` renders a PNG and returns it inline; choose the base **deliberately**
via `purpose:'plot'` (satellite — what is on the land), `'terrain'` (JUPEM 1:50k relief),
`'context'` (street + labels), `'pair'` (both, for side-by-side recon); explicit
`satellite`/`topo`/`views`/`labels` override the preset and the response's `basemap` block
reports what actually applied — cite it. `layers:[ids]` draws snapshot layers under your own
markup (≤3 layers × 150 features; they query the **rendered viewport**, so zoom decides
coverage). `grid:true` + `scale_bar:true` give georeferenced output (bbox + metres/pixel +
pixel↔lat/lon transform in the JSON — read coordinates off the grid and feed them to
`measure`). Image ≤1024×1024. `imagery_info` satellite capture date/resolution without
rendering (cite it with any "measured on the map" claim); `imagery_history` which Esri
Wayback releases (2014→today) carry imagery at the point + a tile template.

**MEASURE & ROUTE** — `measure` exact geodesic distance/path length, or area (m², ha) +
perimeter of a closed polygon — use it instead of estimating off pixels; `route` road
distance/time + geometry (OSRM **car profile only** — fine for planning, not HGV weight/
height aware); `sea_route` indicative nautical distance between ports + passages crossed
(MARNET network, coarse); distances without road routing are straight-line unless stated.

**AGGREGATE & STATUS** — `site_report` one call = the whole point recon, grouped the way a
decision maker reads it (land planning, tenure, safety/environment, terrain/imagery,
utilities, marine, property); snapshot layers feed their own group, queried within 1 km,
capped at 3 hits, distances to the feature **CENTROID** (no point-in-polygon — proximity is
not containment). Per-source best-effort: an `{ error }` inside a group never fails the
call. `service_status` probes FMRS, Overpass, OSRM, gisdev, JUPEM UPI and JPPH and reports
which are answering (cached 5 min, `force:true` re-probes) — **the first triage step:
a timeout is not "nothing there" until a probe says the source answered**.

## The agent loop

```
place_detect(lat, lon)                 → the coordinate is anchored to a citable place
layers_list()                          → valid snapshot layer ids + their coverage bbox
construction_site(lat, lon)            → the project lens in one call
layer_query(layer, lat, lon)           → a specific snapshot layer near the point
road_access / nearest / poi_search     → serving road + logistics
static_map(purpose:'plot', layers:[…], grid:true, scale_bar:true) → show it
measure(points)                        → exact numbers for anything you estimated
```

For plain "what is at this point" recon, swap `construction_site` for `whats_here` +
`land_cover` + `land_use` + `forest_status`/`sarawak_land_status` (or just call
`site_report` once). Compare plots: `land_cover_stats` per plot, `measure` each polygon's
area, `route`/`sea_route` between them, one static_map marking both.

## Honesty rules (carry into every answer)

- **A missing value is not a fact.** Blanks and 0 return `null` and are named in `unstated`.
- **No coverage is not evidence of absence.** Each result block carries its own source list
  and error/coverage note — an empty answer from an out-of-area layer says "not covered".
- Extracts are **planning-grade, not survey-grade**; DEM is ±10 m; GEBCO is ≈450 m cells.
- `jpph_comparables` are reference records, **not transactions and not a valuation**.
- Nothing here is a title search, a valuation, a permit or a geotechnical opinion.
- Proximity in snapshot results is to the feature centroid, not containment.
- Keep the attribution that comes with results.

## Gotchas (each has cost a debugging session)

- **Overpass-backed tools (`poi_search`, `nearest`, `road_access`, `features_in_area`,
  `whats_here`) can be slow** on the public instance: on a timeout, retry **once** (the
  query keeps running and caches), then shrink the radius — don't loop.
- FMRS's point-in-polygon queries silently return nothing for a plain 4326 point; the server
  already wraps points in a small envelope — so if `forest_status`/`mukim_lookup` return
  nothing while `service_status` shows FMRS answering, the area simply has no polygon there.
- `land_use` rides a dev server: transient timeouts are normal, retry cleanly; don't treat a
  timeout as "unplanned land".
- `admin_lookup`/`cadastral_lookup` are offline and instant — use them to sanity-check a
  geocode before trusting downstream results.
- JUPEM topo basemap is legible only around z13–15; use it for relief and contours, not
  labels.
- A 503 or an outage across a whole mirror family is possible — `service_status` first.
- Anything a source answers HTTP 200 with an error body (FMRS stopped) surfaces here as an
  explicit `unknown` verdict — never quietly downgraded to a negative finding.
