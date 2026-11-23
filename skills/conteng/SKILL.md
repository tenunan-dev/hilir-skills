---
name: conteng
description: Annotate PDFs and images with mm-level precision through the Conteng MCP server (conteng.hilir.app) — highlights, boxes, arrows, numbered review pins, callouts, stamps, true redaction, page surgery, drawing coordinate-system registration, annotated exports and review reports. Use it whenever the user asks to mark up, annotate, review, comment on, redact, measure or check a PDF, plan, drawing or image; when asked "where is X on the drawing" in plant Easting/Northing; or to produce an annotated deliverable or a numbered review report.
---

# Conteng — annotating documents and drawings over MCP

Conteng is a remote MCP server at `https://conteng.hilir.app/mcp` (25 tools — `tools/list`
is the live count). It stores a document
byte-identically, keeps markup in an annotation store, and composes it onto a copy at
view/export time — **the original is never modified**. Every tool is self-describing: read
the tool's description before guessing a parameter, and trust it over memory.

## Connect

```
URL:     https://conteng.hilir.app/mcp     (stateless streamable HTTP, JSON responses)
Headers: Authorization: Bearer <key>      (or X-Api-Key: <key>)
```

The credential is an **estate agent key** from access.hilir.app (account page → *Agent
keys*), not a conteng-specific one — the same key works at every hilir estate MCP server.
It must carry scope `conteng:annotate`: a 403 `insufficient_scope` or a 401 naming
`/.well-known/oauth-protected-resource` means the key lacks the scope or has expired.

```json
{
  "mcpServers": {
    "conteng": {
      "type": "http",
      "url": "https://conteng.hilir.app/mcp",
      "headers": { "Authorization": "Bearer <estate-key>" }
    }
  }
}
```

## The core loop — always close it

`put_file → info → view → locate/geometry → annotate → view (verify) → export → download → report`

1. **`conteng_put_file`** takes `data_b64` or a fetchable `url`; returns `file_id`, kind,
   page count and per-page pt sizes. Do this before anything else.
2. **`conteng_info`** next: page count/sizes, vector-drawing vs scan hint, existing
   annotation layers/types, registered CRS. It stops you duplicating earlier markup.
3. **Anchor precisely.** `conteng_locate` (text → bounding box in pt) and `conteng_geometry`
   (vector lines; `snap_point` → exact vertex/edge with distance) give exact anchors. Never
   eyeball a coordinate off a picture when a locate hit or a snap exists.
4. **`conteng_annotate`** a batch of items (type table in the tool description). Every
   item carries its **own `units`** — coordinates straight from `locate`/`geometry` are
   `pt`, so pass `units: "pt"` with them; the default is `fraction` (0–1) and mixing them
   up is the single most common 400.
5. **`conteng_view` again — the verify step is not optional.** Check placement before
   exporting. If a mark is off, prefer `conteng_edit_annotation` (keeps id and pin number)
   over remove + re-add.
6. **Deliver.** `conteng_export` → `conteng_download(export_id)` → `conteng_report` when
   pins/notes/callouts exist. Verify `sha256` from the export against the downloaded bytes.

An MCP client has no HTTP key, so `download_path` is out of reach: `conteng_download`
returns the deliverable as `content_b64` — decode and write it to disk. Payload is capped
at 8 MB; above that the error names the human viewer link (`/f/{id}?t=<share-token>`,
capability URL, no key needed) as the alternative.

## Coordinates

Pages are **0-based** everywhere: `page` parameters, results, and the report's `page`
(the report also prints a 1-based `page_label`).

| Units | Meaning | Needs |
|---|---|---|
| `fraction` (default) | 0–1 across the page | — vision-only, ±1–3 mm |
| `pt` | page points | exact — what locate/geometry return |
| `mm` | paper millimetres from top-left | exact |
| `ground_m` | ground metres from top-left | a scale (`conteng_set_scale` or a CRS) |
| `en` | plant Easting/Northing | a registered CRS (`conteng_grid` / `conteng_calibrate`) |

- On a plan with a labelled border ruler/grid, register once with **`conteng_grid`**: ≥2
  vertical lines carrying Easting values + ≥2 horizontal lines carrying Northing values
  (Northing must grow upward), `zones` for border-ruler areas, `declared_scale` if the
  sheet states one. The *measured* grid scale becomes the file scale — on disagreement you
  get `scale_conflict` and **the measurement wins**.
- Scans/photos without a readable grid: **`conteng_calibrate`** (points ≥2 known references,
  or scale_origin). Residual > 1 mm marks the registration INVALID — re-read better
  reference points instead of trusting it.
- Scale only, no E/N: **`conteng_set_scale`** — then `units: "ground_m"` works.
- **`conteng_where`** converts both ways (`en=[E,N]` → page position for annotating; a
  located label → its E/N). Run it before annotating with datasheet coordinates. It is a
  **diagnostic converter**: for a position off the sheet it still answers, adding
  `off_page: true` and `page_size_pt` ("the flare sits 200 m off the drawing" is a fact, not
  an error). The write tools — `annotate`, `grid` — are the ones that reject off-page input.
- **Coordinates may be a list or a numeric string.** `[120.5, 340]` and `"120.5, 340"` both
  work for any pair or quad (`point`, `en`, `rect`, `at`, `points`, `region`, `crop`); pass
  `units` alongside, since a string does not carry them.

## Markup types — pick by intent

| Intent | Type |
|---|---|
| Text emphasis | highlight / underline / squiggly / strikeout (`target_text` located for you, or boxes) |
| Spatial feedback | box / ellipse, arrow / line |
| Review comments | **pin** (auto-numbered, feeds the report), note, callout |
| Boundaries / freehand | polygon / polyline, ink |
| Approvals | stamp / image_stamp |
| Content must vanish | **redact** — permanently removed at PDF export; opaque fill on images |

Start each review round on its own **layer** (`conteng_layers`); exports and reports can
include one layer, or `"all"`.

## Traps

- **`conteng_export` redacts for real.** Exported copies lose redacted content permanently;
  the stored original and annotation history survive and re-export fine.
- **Exports and files accumulate server-side.** After delivering, clean up:
  `conteng_delete_export` removes the artifact only (annotations and original survive — it
  is simply re-exportable); `conteng_delete_file` deletes the file *and* its annotations,
  layers, CRS and exports permanently.
- **Find a handle you lost.** `conteng_files` lists every file in the workspace (id, name,
  page count, annotation count); `conteng_list_exports` inventories one file's exports;
  `conteng_rename_file` gives a file a meaningful display name without touching its content,
  annotations or id.
- **Page surgery yields a NEW file with no annotations.** `conteng_pages` and
  `conteng_new_from` never modify the original; existing markup stays attached to it, page
  indices may shift, and the copy must be annotated afresh. Within one `conteng_pages` call
  the ops run as a **sequence**: each is validated against the page count the previous op
  left, so `[duplicate p0, insert_blank p1]` on a 1-page file gives 3 pages, and
  `insert_blank` may name `page == page_count` to append.
- **A PDF keeps at least one page.** `conteng_pages` refuses to delete the last remaining
  page (`400 delete: page 0 is the only page — … use conteng_delete_file`), because a
  zero-page document cannot be saved. To discard a document, use `conteng_delete_file`;
  page surgery is for trimming, not for emptying.
- **Raster images have no text layer** — `conteng_locate` and `conteng_text` return
  nothing. Use `conteng_view` + your own vision, and `conteng_calibrate` for precision;
  `conteng_geometry` works on vector PDFs only.
- **DOCX/XLSX/PPTX are unsupported** (phase 2). XPS/EPUB/MOBI/FB2/CBZ/SVG annotate via a
  converted PDF copy.
- **Check `conteng_list_annotations` before a markup pass** — editing an existing mark beats
  duplicating one, and ids come only from that listing.
- **`conteng_view` renders ≤10 pages per call**; use `crop` (fraction rect) to zoom borders,
  grid labels and fine details before registering a CRS.
