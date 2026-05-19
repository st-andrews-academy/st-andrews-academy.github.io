# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

St. Andrews Academy Alumni Portal — a single-file SPA for the Batch 1986 alumni of St. Andrews Academy. All logic, styles, seed data, and embedded images live in one file: **`index.html`**.

**Live site:** https://st-andrews-academy.github.io  
**Admin PIN default:** `aneza654457`

## Running the App

No build step. Open `index.html` directly in a browser, or deploy to GitHub Pages by uploading the file to the `st-andrews-academy/st-andrews-academy.github.io` repository (goes live in ~2-5 minutes).

There are no package managers, bundlers, linters, or test frameworks.

## Architecture

### Single-File Structure

All HTML, CSS (in `<style>`), and JavaScript (in `<script>`) are in `index.html`. Images (class portraits, historical photos) are embedded as base64 data URIs, which accounts for the large file size.

### Navigation & Tabs

`showPage(id, btn)` controls which of the 7 tabs is visible: Overview, History, Graduation, Alumni DB, Faculty, Memory Lane, Settings. Each tab has an `init*()` function called on `DOMContentLoaded`.

**Dashboard-specific reunion sections:**
- **Memory Lane** — stat bar (`#mlReunionStats`) shows 2026 Reunion Grand Total (₱225,200) and Contributors (98), rendered once by `renderEvents()`
- **Faculty** — "2026 Reunion — Teacher Attendance" card (`#facReunionList`) lists attending/absent teachers, rendered by `initFaculty()`
- These sections do not appear on any other dashboard

**Overview page order (top to bottom):**
1. Hero banner
2. "About St. Andrew's Academy" card
3. Stats bar (`#overviewStats`) — Total Alumni, Female, Male, Batches, Sections, Years of Service
4. Two-column: Section Summary table + Batch 1986 Statistics

**Alumni DB page:** shares the same stats bar HTML as Overview (`#dbStats`), populated by the same `statsHtml` variable in `initOverview()`.

### Data Layer (localStorage)

All data persists client-side only — there is no server or backend.

| localStorage key | Contents |
|---|---|
| `saa_alumni` | Alumni records array |
| `saa_faculty` | Faculty records array |
| `saa_events` | Memory Lane events array |
| `saa_settings` | `{ fontSize }` |
| `saa_pending_reg` | Pending registrations array |
| `saa_admin_pin` | Custom PIN (falls back to default if absent) |

On first load, `saa_alumni` and `saa_faculty` are populated from the embedded `ALUMNI_SEED` and `FAC_SEED` arrays. Changes are saved via `saveDB()`, `saveFacDB()`, and `saveEvents()`.

### Core Data Shapes

```js
// Alumni
{ Student_Code, First_Name, Middle_Initial, Surname, Married_Surname,
  Section, Batch, Gender, Profession, Birthday, Marital_Status,
  Children_Information, Deceased, Remarks }

// Faculty
{ Faculty_Code, Surname, First_Name, Middle_Initial, Gender, Position,
  Specialization, Year_Start, Year_End, Employment_Status,
  Batches_Advised, Sections_Advised, Birthday, Attended_2026_Reunion, Remarks }

// Event (Memory Lane)
{ title, date, desc, photos: [base64DataURL, ...], created }

// Pending Registration
{ First_Name, Middle_Initial, Surname, Married_Surname, Section, Batch,
  Gender, Birthday, Profession, Contact, Message, Submitted, Status }
```

### Global State

```js
let db;              // Alumni array
let facDB;           // Faculty array
let events;          // Memory Lane events
let settings;        // { fontSize }
let currentFontSize;
let sortCol, sortDir;     // Alumni table sort state
let facSortCol, facSortDir;
let currentPage;     // Alumni table pagination
let _pinCb;          // Pending PIN-protected callback
```

### PIN Authentication Pattern

Write operations are PIN-protected. `pinThen(fn)` shows the PIN modal and calls `fn()` on success. To add a new PIN-protected action, wrap it: `pinThen(() => yourAction())`.

### Modal Pattern

All modals share a consistent structure: an overlay `<div>` toggled with `display:flex/none`, a close button, and a form that calls a `save*()` function. See `openAddModal()` / `saveStudent()` for the canonical example.

### Seed Data Updates

To update the default dataset visible to all users on first load (or after clearing localStorage), edit the `ALUMNI_SEED` or `FAC_SEED` arrays in `index.html`, then redeploy. Existing users with data in localStorage will not see seed changes unless they restore from a backup or clear their storage.

### Embedded Images

`BATCH_CLASS_PHOTOS` maps section letter → base64 PNG (class portraits). `HIST_IMGS` is a key→base64 map for history photos; `HIST_PHOTOS` is the ordered metadata array (`{key, cap}`) that controls what is displayed and labeled in the gallery. To relabel a photo edit its `cap`; to hide one remove its entry from `HIST_PHOTOS` (leave `HIST_IMGS` intact to avoid corrupting adjacent base64 data). Adding or replacing images means replacing the base64 string directly in the file, which will further increase file size.

The gallery is fully responsive — CSS Grid with `auto-fill` and `minmax(180px, 1fr)` automatically reflows to fewer columns on smaller screens (typically 2 on mobile, 1 on very small screens). No extra work needed when adding photos.

**Current History gallery (16 photos, keys h001–h017, h009 removed as duplicate):**

| Key | Caption |
|---|---|
| h001 | SAA School Seal - circa 1980s |
| h002 | Historical Photo - circa 1910 |
| h003–h005 | School Building |
| h006–h008 | Faculty Members |
| h010–h016 | Class Photo |
| h017 | School ID - circa 1980s |

## Key Functions Reference

| Function | Purpose |
|---|---|
| `renderTable()` | Rebuild alumni table with current filters/sort/page |
| `getFiltered()` | Returns filtered+sorted alumni array |
| `sortTable(col)` | Toggle sort on column |
| `openAddModal()` / `saveStudent()` | Add alumni |
| `editStudent(idx)` / `deleteStudent(idx)` | Edit/delete alumni |
| `renderFaculty()` | Rebuild faculty table |
| `showBatch(year)` / `showClass(year, sec)` | Graduation tab navigation |
| `renderEvents()` / `saveEvent()` | Memory Lane list + add |
| `backupData()` / `restoreData()` | JSON backup/restore |
| `exportCSV()` | Download alumni as CSV |
| `toast(msg)` | Show a brief notification |
| `dl(name, content, type)` | Trigger a file download |
| `genCode(batch, sec)` | Auto-generate student code |
