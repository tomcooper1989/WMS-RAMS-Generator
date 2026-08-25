# WMS RAMS Generator

Single-page web app that produces **RAMS documents** (Risk Assessment & Method Statement)
for WMS underfloor-heating installations. Everything runs **client-side** — there is no
backend. The user fills an 8-step wizard and the app generates a full RAMS **PDF** in the
browser with jsPDF, plus a `.json` project file so a RAMS can be reloaded and amended later
("Load previous RAMS" on step 1).

## Layout

The entire app is **one file: `index.html`** (~2,165 lines, ~3.9 MB).
Most of that size is the inlined **jsPDF** library and ~13 base64 `data:` images
(logo, system cross-section diagrams). The hand-written app code is roughly lines
**520–2165**; everything before that is the vendored library.

### Wizard steps (sections `#sec0`–`#sec7`)
`01 Project` → `02 System` → `03 Personnel` → `04 Site` → `05 Hospital (nearest A&E)`
→ `06 Risk assessment` → `07 PPE` → `08 Review & generate`

### Key functions
| Function | Role |
|---|---|
| `SYSTEM_DATA` | Per-system content: name, scope of works, sequence of operations, diagram |
| `collectData()` | Reads every form field into one `d` object. Falls back to `[Placeholders]` |
| `buildSummary()` / `buildPreview()` | Step-8 summary grid + on-screen HTML preview |
| `downloadPDF()` | **The big one** — builds the entire PDF, then `doc.save(fname)` |
| `raHeader()` / `raRow()` | Render the construction risk-assessment tables (inside `downloadPDF`). `raRow()` shades the "Risk Level Before/After Controls" cells by band (see Risk-assessment colour key below) |
| `riskBand(score)` | Maps a risk score to its band + colour: 1-4 Low `#00B050`, 5-9 Moderate `#FFFF00`, 10-14 Considerable `#FFC000`, 15-19 High `#FF0000`, 20-25 Critical `#C00000`. Shared by `raRow()` and the printed key so they can't drift apart |
| `milesBetween()` | Haversine distance (added 2026-07-24, replaced a flat-Euclidean-degrees calc that overstated distance ~61% at UK latitudes) — used for nearest-A&E sorting/mileage |
| `escHtml()` | Escapes user text before it goes into the on-screen preview's `innerHTML` (added 2026-07-24) |

### Risk-assessment colour key (added 2026-07-24, to match a client-supplied format)
The RA table's two computed score columns are shaded by `riskBand()`, and a full coloured key/legend
(severity + likelihood description tables, colour-swatched band legend, 5×5 risk matrix) is printed
to match a reference document's page-18 key. Laid out as two columns since the app is portrait and
the reference was landscape — same colours/numbers, different arrangement.

### Autosave (added 2026-07-24)
Form data (including PPE/system card selections) is saved to `localStorage` as you type (~1s debounce)
and restored on reload with a banner + "Start fresh" button. Silently no-ops in private/incognito
windows. Persists even after a PDF download — only "Start fresh" clears it.

### PDF conventions (inside `downloadPDF`)
A4 portrait, units in **mm**: `M=18` (margin), `PW=210`, `CW=174` (content width).
Helpers: `np(need)` page-break, `h1/h2/p` headings & paragraphs, `trow` table row,
`sanitise()` strips non-ASCII (jsPDF's helvetica can't render them).

## Deploy

```
edit index.html  →  git commit  →  git push origin main
```
Repo: **github.com/tomcooper1989/WMS-RAMS-Generator** (public, branch `main`).
Railway auto-deploys on push to `main`. There is no build step — it serves `index.html`.

## Testing PDF output locally

The PDF is generated in the browser, so you must actually **look at the rendered pages** —
text extraction will not reveal layout bugs like overlapping columns.

1. Serve the folder:
   `py -m http.server 8099 --directory C:\Users\TomCooper\wms-rams-tool`
   ⚠️ **Do not use port 8765** — that belongs to the separate WMS Pricing app.
2. In the page, call `downloadPDF()`, intercept `URL.createObjectURL` to capture the
   `application/pdf` blob, and POST the bytes to a small local receiver that writes a file.
   (The in-app browser sandboxes downloads — they do not reach the OS Downloads folder.)
3. Rasterise and inspect:
   ```python
   import fitz                      # PyMuPDF IS installed; poppler is NOT
   d = fitz.open('out.pdf')
   d[5].get_pixmap(matrix=fitz.Matrix(2, 2)).save('p6.png')
   ```
   The Read tool's built-in PDF rendering needs poppler and will fail — use PyMuPDF.

## In progress — not live

**Liam Iddon's RAMS content update** (branch `liam-rams-update`, last touched 2026-07-24, **not pushed
to GitHub** — check `git branch -a` / `git log liam-rams-update` before assuming it's gone). Liam
(WMS UK) wants his RA format/content matched exactly; this branch has AmbiTak's Scope/Sequence
rewritten to his wording plus the colour-coded RA cells and key above, as a sample only. Tom said
explicitly not to merge to `main` until he's reviewed it — sample PDFs are at
`Desktop\AI Devs\2.  RAMS\SAMPLE - AmbiTak RAMS v2 (with key - NOT LIVE).pdf`. Other systems and
Liam's promised screeding/concrete RAMS updates are not yet incorporated. Full details in the
`liam-rams-content-update-in-progress` memory file.

## Gotchas

- **Don't trust copies of `index.html` outside this repo.** Dozens of stale duplicates exist across
  Tom's Desktop/Downloads/OneDrive with misleading names/dates. Verify by content hash against this
  repo's `main`, not by filename — see the `stale-duplicate-html-copies` memory file for a concrete
  example of this causing confusion.

- **Duplicated risk-assessment section (fixed 2026-07-21, commit `142e7c3`).** The RA content
  was coded **twice**. The first block drew the *Hazard / Affected / S* cells as single
  **unwrapped** lines, so long text overflowed into neighbouring columns and printed on top of
  them. The fix deleted that duplicate, keeping the correct "Section 28" renderer.
  **Lesson:** `raRow()` must wrap *every* column with `splitTextToSize(txt, g[i]-2)` and set
  row height from the tallest cell — never draw a cell as one unwrapped line.
- **Column widths must sum to `CW`.** `g = [7,22,30,28,6,6,6,CW-124,6,6,7]`. The control-measures
  column absorbs the remainder; if you change any fixed width, fix the `CW-124` term too.
- **This is a separate project** from the WMS Pricing app
  (`…/Desktop/AI Devs/4. Pricing Sheet Project`) and the BOM tool (`~/wms-ufh-bom-tool`).
  Keep them apart — don't run servers for one from inside another.
