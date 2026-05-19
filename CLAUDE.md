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

`BATCH_CLASS_PHOTOS` maps section letter → base64 PNG (class portraits). `HIST_PHOTOS` is an array of base64 JPEGs for the History tab. Adding or replacing images means replacing the base64 string directly in the file, which will further increase file size.

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
