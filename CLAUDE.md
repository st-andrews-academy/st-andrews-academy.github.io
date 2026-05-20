# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

St. Andrews Academy Alumni Portal — a single-file SPA for the Batch 1986 alumni of St. Andrews Academy. All logic, styles, seed data, and embedded images live in one file: **`index.html`**.

**Live site:** https://st-andrews-academy.github.io  
**Admin PIN default:** `aneza654457`

## Running the App

No build step. Open `index.html` directly in a browser, or deploy to GitHub Pages by uploading the file to the `st-andrews-academy/st-andrews-academy.github.io` repository (goes live in ~2-5 minutes).

Firestore requires an origin allowed in Firebase; running from `file://` may fail CORS — use a local HTTP server (e.g. `python -m http.server`) or just open from GitHub Pages. There are no package managers, bundlers, linters, or test frameworks in the project itself.

## Testing

No test suite is checked in. For automated browser testing against the live site, use **Playwright** (available via `npx playwright`):

```bash
npm install playwright        # install locally (don't commit node_modules)
npx playwright install chromium
node my_test.mjs              # run tests
rm -rf package.json package-lock.json node_modules my_test.mjs  # clean up
```

**Key patterns for writing tests:**
- Use `waitUntil: 'load'` (not `networkidle`) — Firebase keeps connections open and `networkidle` never fires.
- Wait for `#loadingOverlay` to be hidden before interacting: `page.waitForSelector('#loadingOverlay', { state: 'hidden' })`.
- Modals are closed with explicit Cancel buttons, not Escape — use `page.click('#modalId button:has-text("Cancel")')`.
- After cancelling the PIN modal, the parent modal is still open — close it separately.
- The admin PIN for testing is `aneza654457` (default); read it from `ADMIN_PIN_DEFAULT` in the source.

**Live URLs under test:**

| Target | URL |
|---|---|
| Alumni Portal | `https://st-andrews-academy.github.io` |
| Architecture Diagram | `https://st-andrews-academy.github.io/SAA_Architecture_Diagram.html` |

**Architecture diagram specifics:** The diagram draws SVG arrows via JavaScript on load — add `page.waitForTimeout(1000)` after page load before asserting arrow counts. Key assertions: 8 boxes visible (`#b-dev` through `#b-mob2`), ≥8 `path.arrow-line` elements in the SVG, all PIN access control items present and matching the app's actual behaviour.

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

### Data Layer

**Primary store: Firestore.** On load, `loadFromFirestore()` fetches all data from Firestore and syncs it into localStorage as a cache. If Firestore is unreachable, the app falls back to localStorage (or seed data on first visit). All writes go to both localStorage and Firestore in parallel — Firestore failures are non-fatal (`console.warn` only).

Firebase SDK v10.12.0 compat is loaded via two CDN `<script>` tags (lines 173–174). The global `fstore` variable holds the Firestore instance.

**Firestore collections / documents:**

| Path | Contents |
|---|---|
| `portal/alumni` | `{ records: [...] }` — full alumni array |
| `portal/faculty` | `{ records: [...] }` — full faculty array |
| `portal/config` | `{ adminPin: "..." }` — admin PIN |
| `portal/pending` | `{ records: [...] }` — self-registration submissions |
| `portal/pending_alumni` | `{ records: [...] }` — pending alumni add submissions |
| `portal/pending_faculty` | `{ records: [...] }` — pending faculty add submissions |
| `portal/pending_events` | `{ records: [...] }` — pending event add submissions |
| `events/{evId}` | Individual event doc; `evId` is `Date.now().toString()` |

Events are stored as individual Firestore documents (not in a single array doc) and carry an `_id` field matching their document ID. `saveEvents()` only writes to localStorage; Firestore writes for events are done explicitly at creation (`fstore.collection('events').doc(evId).set(ev)`), edit (`fstore.collection('events').doc(ev._id).set(ev)`), and deletion (`fstore.collection('events').doc(ev._id).delete()`).

**localStorage keys (cache / fallback):**

| Key | Contents |
|---|---|
| `saa_alumni` | Alumni records array |
| `saa_faculty` | Faculty records array |
| `saa_events` | Memory Lane events array |
| `saa_settings` | `{ fontSize }` |
| `saa_pending_reg` | Self-registration submissions array |
| `saa_pending_alumni` | Pending alumni add submissions array |
| `saa_pending_faculty` | Pending faculty add submissions array |
| `saa_pending_events` | Pending event add submissions array |
| `saa_admin_pin` | Custom PIN (falls back to default if absent) |

On first load, if Firestore has no data, `ALUMNI_SEED` and `FAC_SEED` are written to both Firestore and localStorage. Existing users see Firestore data on every load, so seed changes in the file are **ignored** for existing users unless Firestore data is cleared.

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
let sortCol='Surname', sortDir=1; // Alumni table sort state
let currentPage=1, pageSize=25;   // Alumni pagination (25 rows/page)
let editIdx=-1;                   // -1 = add mode, ≥0 = edit mode for alumni modal
let facSortCol='Surname', facSortDir=1;
let facPage=1, facPageSize=20;    // Faculty pagination (20 rows/page)
let facEditIdx=-1;                // -1 = add mode, ≥0 = edit mode for faculty modal
let _pinCb;          // Pending PIN-protected callback
let editEventIdx;    // -1 = add mode, ≥0 = edit mode for event modal
let pendingPhotos;   // File[] accumulated across multiple file-picker opens
let existingPhotoUrls; // base64[] of photos already saved (edit mode only)
let _pendingObjectUrls; // object URLs created for photo thumbnails — revoked on each redraw and on modal close
let _toastTimer;     // clearTimeout handle so rapid toasts don't race
```

### PIN Authentication Pattern

`pinThen(fn)` shows the PIN modal and calls `fn()` on success. To add a new PIN-protected action, wrap it: `pinThen(() => yourAction())`.

**Add → Pending queue (no PIN, open to anyone):**
- **Add alumni** — `submitPendingAlumni()` saves to `saa_pending_alumni` / `portal/pending_alumni` for admin approval.
- **Add faculty** — `submitPendingFaculty()` saves to `saa_pending_faculty` / `portal/pending_faculty` for admin approval.
- **Add event** — `submitPendingEvent()` saves to `saa_pending_events` / `portal/pending_events` for admin approval.
- The Save button in add modals shows "📝 Submit for Approval" (no PIN badge). `saveStudentWithPin` / `saveFacultyWithPin` / `saveEventOrSubmit` route to the submit path when `editIdx < 0` / `facEditIdx < 0` / `editEventIdx < 0`.

**PIN required — Admin only:**
- **Edit** alumni/faculty — PIN required **before** opening the edit modal (`editStudentPin` / `editFacultyPin` → `pinThen`). No second PIN on Save. Save button shows "💾 Save Record PIN".
- **Edit event** — `openEditEventModal(i)` opens freely; PIN on Save (`saveEventOrSubmit` → `pinThen(()=>saveEvent())` when `editEventIdx >= 0`). Save button shows "💾 Save Event PIN".
- **Delete** alumni / faculty / event — `deleteStudentPin` / `deleteFacultyPin` / `deleteEventPin` → `pinThen`.
- **Approve / Reject** any pending submission (alumni add, faculty add, event, self-registration) — all route through `pinThen`.
- **Restore** alumni or faculty JSON backup — `pinThen(restoreData)` / `pinThen(restoreFaculty)`.

**Pending Approvals (Settings tab):** Four queues shown in `#pendingList`:
1. Pending Alumni Additions (`saa_pending_alumni`) — `approvePendingAlumni` / `rejectPendingAlumni`
2. Pending Faculty Additions (`saa_pending_faculty`) — `approvePendingFaculty` / `rejectPendingFaculty`
3. Pending Events (`saa_pending_events`) — `approvePendingEvent` / `rejectPendingEvent`
4. Self-Registrations (`saa_pending_reg`) — `approvePending` / `rejectPending`

`updatePendingBadge()` sums all four queues for the Settings tab badge.

**Not PIN-protected (open to anyone):** Opening any Add modal, submitting Add forms (goes to pending), save display settings (`saveSettings`), backup/export downloads, self-registration submission.

**Change Admin PIN** uses its own modal (`openChgPin` / `doChgPin`) that requires entering the current PIN directly — it does not use `pinThen()`.

### Modal Pattern

All modals share a consistent structure: an overlay `<div>` toggled with `display:flex/none`, a close button, and a form that calls a `save*()` function. See `openAddModal()` / `saveStudent()` for the canonical example.

### Seed Data Updates

To update the default dataset visible to all users on first load (or after clearing localStorage), edit the `ALUMNI_SEED` or `FAC_SEED` arrays in `index.html`, then redeploy. Existing users with data in localStorage will not see seed changes unless they restore from a backup or clear their storage.

Seed counts: **321 alumni** across 7 sections (A–G), **15 faculty** records.

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
| `openAddModal()` / `saveStudentWithPin()` | Open Add Alumni modal / route to `submitPendingAlumni` (add) or `saveStudent` (edit) |
| `submitPendingAlumni()` | Submit new alumni to pending queue (no PIN) |
| `editStudentPin(idx)` / `editStudent(idx)` | Edit alumni — PIN before opening modal |
| `deleteStudentPin(idx)` / `deleteStudent(idx)` | Delete alumni (PIN-protected) |
| `approvePendingAlumni(i)` / `rejectPendingAlumni(i)` | Approve/reject pending alumni submission (PIN-protected) |
| `renderFaculty()` | Rebuild faculty table |
| `openAddFacultyModal()` / `saveFacultyWithPin()` | Open Add Faculty modal / route to `submitPendingFaculty` (add) or `saveFaculty` (edit) |
| `submitPendingFaculty()` | Submit new faculty to pending queue (no PIN) |
| `editFacultyPin(idx)` / `editFaculty(idx)` | Edit faculty — PIN before opening modal |
| `deleteFacultyPin(idx)` / `deleteFaculty(idx)` | Delete faculty (PIN-protected) |
| `approvePendingFaculty(i)` / `rejectPendingFaculty(i)` | Approve/reject pending faculty submission (PIN-protected) |
| `showBatch(year)` / `showClass(year, sec)` | Graduation tab navigation |
| `renderEvents()` | Memory Lane list render |
| `openEventModal()` / `openEditEventModal(i)` | Open add (→ pending) / edit (→ PIN on save) event modal |
| `saveEventOrSubmit()` | Route to `submitPendingEvent` (add) or `pinThen(saveEvent)` (edit) |
| `submitPendingEvent()` | Submit new event to pending queue (no PIN) |
| `saveEvent()` | Save edited event to Firestore (called via `pinThen`) |
| `approvePendingEvent(i)` / `rejectPendingEvent(i)` | Approve/reject pending event submission (PIN-protected) |
| `appendPhotos(input)` | Accumulate file picks into `pendingPhotos[]` |
| `updatePhotoPreview()` | Render photo thumbnails with ✕ remove buttons |
| `renderPending()` | Render all 4 pending queues in `#pendingList` (Settings tab) |
| `updatePendingBadge()` | Sum all 4 pending queues for the Settings tab badge count |
| `savePending(p)` | Save self-registrations to localStorage + `portal/pending` |
| `savePendingAlumni(p)` | Save pending alumni to localStorage + `portal/pending_alumni` |
| `savePendingFaculty(p)` | Save pending faculty to localStorage + `portal/pending_faculty` |
| `savePendingEvents(p)` | Save pending events to localStorage + `portal/pending_events` |
| `backupData()` / `restoreData()` | JSON backup/restore (restore is PIN-protected) |
| `exportCSV()` | Download alumni as CSV |
| `toast(msg)` | Show a brief notification |
| `dl(name, content, type)` | Trigger a file download |
| `genCode(batch, sec)` | Auto-generate student code |
| `loadFromFirestore()` | Fetch all data (incl. all 4 pending queues) from Firestore on startup; falls back to localStorage |
| `compressImage(file, maxW, quality)` | Client-side canvas compress before storing photo as base64 (default 1200px, 0.72 quality) |
| `changeFont(d)` | Increment/decrement font size (range 10–24px, default 15); writes to `--fs` CSS custom property and localStorage |
| `openLightbox(src)` | Open full-screen photo lightbox |
| `backupFaculty()` | Download faculty JSON backup |
