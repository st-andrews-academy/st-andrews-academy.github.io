# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

St. Andrews Academy Alumni Portal — a single-file SPA for the Batch 1986 alumni of St. Andrews Academy. All logic, styles, seed data, and embedded images live in one file: **`index.html`**.

**Live site:** https://st-andrews-academy.github.io  
**Admin PIN default:** encoded in source (`ADMIN_PIN_DEFAULT`)

**Repo files:**
- `index.html` — the entire portal (only file that matters for deployment)
- `SAA_Architecture_Diagram.html` — standalone architecture diagram, also deployed to GitHub Pages; draw SVG arrows on load via JS
- `Files/` — gitignored directory for local backup exports (JSON, CSV); not part of the codebase

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
- The admin PIN for testing is the decoded value of `ADMIN_PIN_DEFAULT` in the source (`atob(...)`).

**Live URLs under test:**

| Target | URL |
|---|---|
| Alumni Portal | `https://st-andrews-academy.github.io` |
| Architecture Diagram | `https://st-andrews-academy.github.io/SAA_Architecture_Diagram.html` |

**Architecture diagram specifics:** The diagram draws SVG arrows via JavaScript on load — add `page.waitForTimeout(1000)` after page load before asserting arrow counts. Key assertions: 8 boxes visible (`#b-dev` through `#b-mob2`), ≥8 `path.arrow-line` elements in the SVG, all PIN access control items present and matching the app's actual behaviour.

## Architecture

### Single-File Structure

All HTML, CSS (in `<style>`), and JavaScript (in `<script>`) are in `index.html`. Images (class portraits, historical photos) are embedded as base64 data URIs, which accounts for the large file size.

A `Content-Security-Policy` meta tag in `<head>` restricts resource origins: scripts only from `'self'`, `'unsafe-inline'`, and `gstatic.com`; fonts from `fonts.gstatic.com`; connections to `*.googleapis.com`, `*.google.com`, and `*.firebaseio.com`; images from `data:` and `blob:`; no `<frame>` or `<object>`. This blocks most injected-script and data-exfiltration attacks without touching Firebase or Google Fonts.

All dynamic HTML rendering uses `escapeHtml(s)` (defined near the top of the `<script>`) to prevent XSS. It HTML-encodes `&`, `<`, `>`, `"`, and `'`. Applied to every field rendered into the Alumni DB table, Faculty table, Graduation class roster, Overview stats, Memory Lane event cards, event detail view, and Faculty adviser/reunion lists.

### Navigation & Tabs

`showPage(id, btn)` controls which of the 8 tabs is visible: Overview, History, Graduation, Alumni DB, Faculty, Memory Lane, Notices, Settings. Each tab has an `init*()` function called on `DOMContentLoaded`.

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

**Notices page layout (split-panel):**
- The page uses `display:flex; flex-direction:column; height:calc(100vh - 92px); overflow:hidden` so it fills the viewport below the fixed nav and online banner.
- **Announcements card** (top, `flex:2` ≈ 2/3 of height) — independently scrollable list (`#announcementList`, class `notices-list`). Live search via `#annSearch` filters by title or body text in real time (`oninput="renderAnnouncements()"`).
- **Feedback card** (bottom, `flex:1` ≈ 1/3 of height) — independently scrollable list (`#feedbackList`, class `notices-list`). Live search via `#fbSearch` filters by name or message text in real time (`oninput="renderFeedbackList()"`).
- Both cards have CSS class `notices-card` (`display:flex; flex-direction:column; min-height:0; margin-bottom:0`). When no results match a search term, a contextual "No … match your search." message is shown instead of the default "No … yet." message.

**Settings page boxes (`.setting-group` cards, rendered in a CSS grid):**
1. **Backup All & Restore All** — highlighted card (gold border); `backupAll()` produces `SAA_Full_Backup_<date>.json` with all six databases; `restoreAll()` / `doRestoreAll()` restores whichever keys are present and re-renders all affected views
2. Database Backup & Restore — alumni (`backupData` / `restoreData`)
3. Export & Import — alumni CSV/JSON
4. Display Settings
5. Faculty Backup & Restore — `backupFaculty()` / `restoreFaculty()` (also accessible from Faculty toolbar)
6. Memory Lane Backup & Restore — `backupMemoryLane()` / `restoreMemoryLane()`; restore re-creates individual Firestore event docs
7. Notices Backup & Restore — `backupNotices()` / `restoreNotices()`; single file covers both Announcements and Feedback
8. Admin PIN Protection
9. Pending Approvals (`#pendingList`)
10. Portal Information — school details and version stamp
11. **Disclaimer** — four-paragraph static notice stating the site is an independent alumni initiative, not the school's official website, with IP/ownership acknowledgment and a contact prompt for content concerns

### Data Layer

**Primary store: Firestore.** On load, `loadFromFirestore()` fetches all data from Firestore and syncs it into localStorage as a cache. Firebase is the sole source of truth — every page load fetches fresh from Firestore and overwrites localStorage before the UI renders. If Firestore fails to load entirely, a persistent red offline banner appears and all writes are blocked via the `_firestoreOnline` flag + `_requireFirestore()` guard (added to every write entry point). Firestore write failures surface a debounced user-visible toast via `_firestoreWarn()` instead of failing silently.

Firebase SDK v10.12.0 compat is loaded via two CDN `<script>` tags (lines 173–174). The global `fstore` variable holds the Firestore instance.

**Firestore collections / documents:**

| Path | Contents |
|---|---|
| `portal/alumni` | `{ records: [...] }` — full alumni array |
| `portal/faculty` | `{ records: [...] }` — full faculty array |
| `portal/config` | `{ adminPin: "..." }` — admin PIN |
| `portal/pending_alumni_registration` | `{ records: [...] }` — self-registration submissions |
| `portal/pending_alumni_add` | `{ records: [...] }` — pending alumni add submissions |
| `portal/pending_alumni_edit` | `{ records: [...] }` — pending alumni edit submissions |
| `portal/pending_faculty_add` | `{ records: [...] }` — pending faculty add submissions |
| `portal/pending_faculty_edit` | `{ records: [...] }` — pending faculty edit submissions |
| `portal/pending_event_add` | `{ records: [...] }` — pending event add submissions |
| `portal/pending_event_edit` | `{ records: [...] }` — pending event edit submissions |
| `portal/pending_feedback_add` | `{ records: [...] }` — pending feedback add submissions |
| `portal/pending_feedback_edit` | `{ records: [...] }` — pending feedback edit submissions |
| `portal/announcements` | `{ records: [...] }` — announcements (admin-only add/edit/delete) |
| `portal/feedback` | `{ records: [...] }` — approved feedback entries |
| `events/{evId}` | Individual event doc; `evId` is `Date.now().toString()` |

Events are stored as individual Firestore documents (not in a single array doc) and carry an `_id` field matching their document ID. `saveEvents()` only writes to localStorage; Firestore writes for events are done explicitly at creation (`fstore.collection('events').doc(evId).set(ev)`), edit (`fstore.collection('events').doc(ev._id).set(ev)`), and deletion (`fstore.collection('events').doc(ev._id).delete()`).

**localStorage keys (cache / fallback):**

| Key | Contents |
|---|---|
| `saa_alumni` | Alumni records array |
| `saa_faculty` | Faculty records array |
| `saa_events` | Memory Lane events array |
| `saa_settings` | `{ fontSize }` |
| `saa_pending_alumni_registration` | Self-registration submissions array |
| `saa_pending_alumni_add` | Pending alumni add submissions array |
| `saa_pending_alumni_edit` | Pending alumni edit submissions array |
| `saa_pending_faculty_add` | Pending faculty add submissions array |
| `saa_pending_faculty_edit` | Pending faculty edit submissions array |
| `saa_pending_event_add` | Pending event add submissions array |
| `saa_pending_event_edit` | Pending event edit submissions array |
| `saa_pending_feedback_add` | Pending feedback add submissions array |
| `saa_pending_feedback_edit` | Pending feedback edit submissions array |
| `saa_announcements` | Announcements array |
| `saa_feedback` | Approved feedback array |
| `saa_admin_pin` | Custom PIN (falls back to default if absent) |

**Firestore Security Rules** (must be set in Firebase Console → Firestore Database → Rules):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /events/{evId} {
      allow read: if true;
      allow write: if true;
    }
    match /portal/{docId} {
      allow read: if true;
      allow write: if docId in [
        'pending_alumni_registration', 'pending_alumni_add', 'pending_alumni_edit',
        'pending_faculty_add', 'pending_faculty_edit',
        'pending_event_add', 'pending_event_edit',
        'pending_feedback_add', 'pending_feedback_edit'
      ];
      allow write: if docId in ['alumni', 'faculty', 'config', 'announcements', 'feedback'];
    }
  }
}
```

> **Critical:** `loadFromFirestore()` reads all 15 Firestore docs in a single `Promise.all`. If any one read is denied by the rules, the entire load fails and the app shows the offline banner. The rules must allow `read: if true` for all portal documents — including `portal/config` and all 9 pending queue docs. Restricting reads on any of those documents breaks the app entirely.

On first load, if Firestore has no data, `ALUMNI_SEED` and `FAC_SEED` are written to both Firestore and localStorage. Existing users see Firestore data on every load, so seed changes in the file are **ignored** for existing users unless Firestore data is cleared.

### Core Data Shapes

```js
// Alumni
{ Student_Code, First_Name, Middle_Initial, Surname, Married_Surname,
  Section, Batch, Gender, Profession, Birthday, Marital_Status,
  Children_Information, Remarks,
  Email, Cellphone }   // admin-only — stored in DB, never shown in dashboard tables

// Faculty
{ Faculty_Code, Surname, First_Name, Middle_Initial, Gender, Position,
  Specialization, Year_Start, Year_End, Employment_Status,
  Batches_Advised, Sections_Advised, Birthday, Attended_2026_Reunion, Remarks,
  Email, Cellphone }   // admin-only — stored in DB, never shown in dashboard tables

// Event (Memory Lane)
{ title, date, desc, photos: [base64DataURL, ...], created }

// Pending Registration
{ First_Name, Middle_Initial, Surname, Married_Surname, Section, Batch,
  Gender, Birthday, Profession, Contact, Message, Submitted, Status }

// Announcement (admin-only add/edit/delete)
{ _id, title, body, created }

// Feedback (approved)
{ _id, name, message, created }

// Pending Feedback Add
{ name, message, Submitted }

// Pending Feedback Edit
{ _id, name, message, Submitted, _originalId, _displayName }
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
let _fsWarnTimer;    // debounce handle for Firestore write-failure toasts (400ms)
let _firestoreOnline; // true only after a successful loadFromFirestore(); gates all writes via _requireFirestore()
let announcements;   // Announcements array (Notices tab)
let feedbackDB;      // Approved feedback array (Notices tab)
let editAnnouncementIdx; // -1 = add mode, ≥0 = edit mode for announcement modal
let editFeedbackIdx;     // -1 = add mode, ≥0 = edit mode for feedback modal
// PIN security state — initialised from sessionStorage so lockout survives page refresh
let _pinFailCount;   // Failed PIN attempts since last success (persisted in sessionStorage)
let _pinLockUntil;   // timestamp (ms) when lockout expires; 0 = not locked (persisted in sessionStorage)
let _adminSessionExpiry; // timestamp (ms) when the current admin session expires (5-min window)
window._ecPhotos;    // base64[] of current event's photos — set by openEvent() so lightbox onclick
                     // references the array by index instead of embedding raw data URIs in HTML
```

### PIN Authentication Pattern

`pinThen(fn)` shows the PIN modal and calls `fn()` on success. To add a new PIN-protected action, wrap it: `pinThen(() => yourAction())`.

**Add → Pending queue (no PIN, open to anyone):**
- **Add alumni** — `submitPendingAlumni()` saves to `saa_pending_alumni_add` / `portal/pending_alumni_add` for admin approval.
- **Add faculty** — `submitPendingFaculty()` saves to `saa_pending_faculty_add` / `portal/pending_faculty_add` for admin approval.
- **Add event** — `submitPendingEvent()` saves to `saa_pending_event_add` / `portal/pending_event_add` for admin approval.
- **Submit feedback** — `submitPendingFeedback()` saves to `saa_pending_feedback_add` / `portal/pending_feedback_add` for admin approval.
- The Save button in add modals shows "📝 Submit for Approval" (no PIN badge). `saveStudentWithPin` / `saveFacultyWithPin` / `saveEventOrSubmit` / `saveFeedbackOrSubmit` route to the submit path when in add mode.

**Edit → Pending queue (no PIN, open to anyone):**
- **Edit alumni** — `editStudentPin(idx)` opens the modal directly (no PIN). Save calls `submitPendingEditAlumni()` which saves to `saa_pending_alumni_edit` / `portal/pending_alumni_edit`. Save button shows "📝 Submit Edit for Approval". The pending record stores `_originalCode` (Student_Code) and `_displayName` so admin can identify which record is being edited.
- **Edit faculty** — `editFacultyPin(idx)` opens the modal directly (no PIN). Save calls `submitPendingEditFaculty()` → `saa_pending_faculty_edit` / `portal/pending_faculty_edit`. Same pattern as alumni.
- **Edit event** — `openEditEventModal(i)` opens freely. Save calls `submitPendingEditEvent()` → `saa_pending_event_edit` / `portal/pending_event_edit`. Pending record stores `_originalId` (event `_id`) and `_displayName`.
- **Edit feedback** — `openEditFeedbackModal(idx)` opens freely. Save calls `submitPendingEditFeedback()` → `saa_pending_feedback_edit` / `portal/pending_feedback_edit`. Pending record stores `_originalId` (feedback `_id`) and `_displayName`.

**PIN required — Admin only:**
- **Add / Edit / Delete announcements** — `openAddAnnouncementModal()` / `openEditAnnouncementModal(idx)` / `deleteAnnouncement(idx)` — all wrapped in `pinThen`. Announcements are written directly (no pending queue).
- **Delete feedback** (from approved list) — `deleteFeedbackPin(idx)` → `pinThen`.
- **Delete** alumni / faculty / event — `deleteStudentPin` / `deleteFacultyPin` / `deleteEventPin` → `pinThen`.
- **Approve / Reject** any pending submission (alumni add, alumni edit, faculty add, faculty edit, event add, event edit, feedback add, feedback edit, self-registration) — all route through `pinThen`.
- **Backup All / Restore All** — `pinThen(backupAll)` / `pinThen(restoreAll)` — full snapshot of all six databases in one file.
- **Individual backups** — `pinThen(backupData)`, `pinThen(backupFaculty)`, `pinThen(backupMemoryLane)`, `pinThen(backupNotices)`, `pinThen(backupSettings)` — available from Settings; alumni and faculty backups also accessible from their respective tab toolbars.
- **Individual restores** — `pinThen(restoreData)`, `pinThen(restoreFaculty)`, `pinThen(restoreMemoryLane)`, `pinThen(restoreNotices)` — each overwrites only its own dataset. All backup functions also guard with `if(!_requireAdmin())` internally.
- **Export** — `pinThen(exportCSV)`, `pinThen(exportFaculty)` — CSV downloads from Alumni and Faculty toolbars.
- **Save display settings** — `pinThen(saveSettings)`.

**Pending Approvals (Settings tab):** Nine queues shown in `#pendingList`:
1. Pending Alumni Additions (`saa_pending_alumni_add`) — `approvePendingAlumni` / `rejectPendingAlumni`
2. Pending Alumni Edits (`saa_pending_alumni_edit`) — `approvePendingEditAlumni` / `rejectPendingEditAlumni`
3. Pending Faculty Additions (`saa_pending_faculty_add`) — `approvePendingFaculty` / `rejectPendingFaculty`
4. Pending Faculty Edits (`saa_pending_faculty_edit`) — `approvePendingEditFaculty` / `rejectPendingEditFaculty`
5. Pending Events (`saa_pending_event_add`) — `approvePendingEvent` / `rejectPendingEvent`
6. Pending Event Edits (`saa_pending_event_edit`) — `approvePendingEditEvent` / `rejectPendingEditEvent`
7. Pending Feedback (`saa_pending_feedback_add`) — `approvePendingFeedback` / `rejectPendingFeedback`
8. Pending Feedback Edits (`saa_pending_feedback_edit`) — `approvePendingEditFeedback` / `rejectPendingEditFeedback`
9. Self-Registrations (`saa_pending_alumni_registration`) — `approvePending` / `rejectPending`

`updatePendingBadge()` sums all nine queues for the Settings tab badge.

**Not PIN-protected (open to anyone):** Opening any Add or Edit modal, submitting Add/Edit forms (goes to pending), self-registration submission, browsing/searching all records.

**Contact details privacy:** The Add/Edit modals for alumni and faculty include `Email` and `Cellphone` fields. These are stored in the record (Firestore + localStorage) and visible when editing, but are **never rendered** in the Alumni DB or Faculty dashboard tables. A yellow `.privacy-note` banner in each form reads: "Contact details (email & cellphone) and other personal information are for information purpose only, and shall not be used nor shared to the public. Please do not fill if you are not comfortable sharing it in this portal."

**Change Admin PIN** uses its own modal (`openChgPin` / `doChgPin`) that requires entering the current PIN directly — it does not use `pinThen()`.

**Brute-force lockout:** Both `pinThen()` (via `confirmPin()`) and `doChgPin()` enforce a 5-attempt lockout. After 5 consecutive wrong PINs, `_pinLockUntil` is set to `Date.now()+60000` and `_pinFailCount` resets to 0. Further attempts are blocked until the timestamp expires. `_pinFailCount` and `_pinLockUntil` are initialised from `sessionStorage` on page load (via `_saveLockout()`) so the lockout survives a page refresh within the same browser session.

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
| `openAddModal()` / `saveStudentWithPin()` | Open Add Alumni modal / route to `submitPendingAlumni` (add) or `submitPendingEditAlumni` (edit) |
| `submitPendingAlumni()` | Submit new alumni to pending add queue (no PIN) |
| `submitPendingEditAlumni()` | Submit alumni edit to pending edit queue (no PIN) — stores `_originalCode` + `_displayName` |
| `editStudentPin(idx)` / `editStudent(idx)` | Open alumni edit modal directly (no PIN required) |
| `deleteStudentPin(idx)` / `deleteStudent(idx)` | Delete alumni (PIN-protected) |
| `approvePendingAlumni(i)` / `rejectPendingAlumni(i)` | Approve/reject pending alumni add (PIN-protected) |
| `approvePendingEditAlumni(i)` / `rejectPendingEditAlumni(i)` | Approve/reject pending alumni edit — finds record by `Student_Code` and replaces it (PIN-protected) |
| `renderFaculty()` | Rebuild faculty table |
| `openAddFacultyModal()` / `saveFacultyWithPin()` | Open Add Faculty modal / route to `submitPendingFaculty` (add) or `submitPendingEditFaculty` (edit) |
| `submitPendingFaculty()` | Submit new faculty to pending add queue (no PIN) |
| `submitPendingEditFaculty()` | Submit faculty edit to pending edit queue (no PIN) — stores `_originalCode` + `_displayName` |
| `editFacultyPin(idx)` / `editFaculty(idx)` | Open faculty edit modal directly (no PIN required) |
| `deleteFacultyPin(idx)` / `deleteFaculty(idx)` | Delete faculty (PIN-protected) |
| `approvePendingFaculty(i)` / `rejectPendingFaculty(i)` | Approve/reject pending faculty add (PIN-protected) |
| `approvePendingEditFaculty(i)` / `rejectPendingEditFaculty(i)` | Approve/reject pending faculty edit — finds by `Faculty_Code` (PIN-protected) |
| `showBatch(year)` / `showClass(year, sec)` | Graduation tab navigation |
| `renderEvents()` | Memory Lane list render |
| `openEventModal()` / `openEditEventModal(i)` | Open add (→ pending) / edit (→ pending on save) event modal |
| `saveEventOrSubmit()` | Route to `submitPendingEvent` (add) or `submitPendingEditEvent` (edit) |
| `submitPendingEvent()` | Submit new event to pending add queue (no PIN) |
| `submitPendingEditEvent()` | Submit event edit to pending edit queue (no PIN) — stores `_originalId` + `_displayName` |
| `saveEvent()` | Save edited event directly to Firestore (called only via `approvePendingEditEvent`) |
| `approvePendingEvent(i)` / `rejectPendingEvent(i)` | Approve/reject pending event add (PIN-protected) |
| `approvePendingEditEvent(i)` / `rejectPendingEditEvent(i)` | Approve/reject pending event edit — finds by `_id`, updates Firestore doc (PIN-protected) |
| `appendPhotos(input)` | Accumulate file picks into `pendingPhotos[]` |
| `updatePhotoPreview()` | Render photo thumbnails with ✕ remove buttons |
| `initNotices()` | Called by `showPage('notices')` — renders announcements and feedback lists |
| `renderAnnouncements()` | Rebuild announcements list in `#announcementList` (latest-first); reads `#annSearch` and filters by title/body if a search term is present |
| `openAddAnnouncementModal()` / `openEditAnnouncementModal(idx)` | Open announcement add/edit modal (both PIN-gated via `pinThen`) |
| `closeAnnouncementModal()` | Close announcement modal |
| `saveAnnouncement()` | Save announcement directly to Firestore (no pending queue) |
| `deleteAnnouncement(idx)` | Delete announcement (PIN-protected) |
| `saveAnnouncements(a)` | Persist announcements to localStorage + `portal/announcements` |
| `renderFeedbackList()` | Rebuild approved feedback list in `#feedbackList` (latest-first); reads `#fbSearch` and filters by name/message if a search term is present |
| `openAddFeedbackModal()` / `openEditFeedbackModal(idx)` | Open feedback add/edit modal (no PIN) |
| `closeFeedbackModal()` | Close feedback modal |
| `saveFeedbackOrSubmit()` | Route to `submitPendingFeedback` (add) or `submitPendingEditFeedback` (edit) |
| `submitPendingFeedback()` | Submit new feedback to pending add queue (no PIN) |
| `submitPendingEditFeedback()` | Submit feedback edit to pending edit queue (no PIN) — stores `_originalId` + `_displayName` |
| `deleteFeedbackPin(idx)` | Delete approved feedback (PIN-protected) |
| `saveFeedbackDB(f)` | Persist approved feedback to localStorage + `portal/feedback` |
| `savePendingFeedback(p)` | Persist pending feedback adds to localStorage + `portal/pending_feedback_add` |
| `savePendingEditFeedback(p)` | Persist pending feedback edits to localStorage + `portal/pending_feedback_edit` |
| `approvePendingFeedback(i)` / `rejectPendingFeedback(i)` | Approve/reject pending feedback add (PIN-protected) |
| `approvePendingEditFeedback(i)` / `rejectPendingEditFeedback(i)` | Approve/reject pending feedback edit — finds by `_id` and updates `feedbackDB` (PIN-protected) |
| `renderPending()` | Render all 9 pending queues in `#pendingList` (Settings tab) |
| `updatePendingBadge()` | Sum all 9 pending queues for the Settings tab badge count |
| `savePending(p)` | Save self-registrations to localStorage + `portal/pending_alumni_registration` |
| `savePendingAlumni(p)` | Save pending alumni adds to localStorage + `portal/pending_alumni_add` |
| `savePendingFaculty(p)` | Save pending faculty adds to localStorage + `portal/pending_faculty_add` |
| `savePendingEvents(p)` | Save pending event adds to localStorage + `portal/pending_event_add` |
| `savePendingEditAlumni(p)` | Save pending alumni edits to localStorage + `portal/pending_alumni_edit` |
| `savePendingEditFaculty(p)` | Save pending faculty edits to localStorage + `portal/pending_faculty_edit` |
| `savePendingEditEvents(p)` | Save pending event edits to localStorage + `portal/pending_event_edit` |
| `backupAll()` / `restoreAll()` / `doRestoreAll()` | Full-site backup/restore — all six databases in one file (PIN-protected) |
| `backupData()` / `restoreData()` / `doRestore()` | Alumni + events + settings JSON backup/restore (PIN-protected) |
| `backupFaculty()` / `restoreFaculty()` / `doRestoreFaculty()` | Faculty JSON backup/restore (PIN-protected) |
| `backupMemoryLane()` / `restoreMemoryLane()` / `doRestoreMemoryLane()` | Memory Lane backup/restore; restore re-creates Firestore event docs (PIN-protected) |
| `backupNotices()` / `restoreNotices()` / `doRestoreNotices()` | Announcements + Feedback backup/restore in one file (PIN-protected) |
| `backupSettings()` | Settings JSON download (PIN-protected; no dedicated restore — settings are also covered by Backup All) |
| `exportCSV()` | Download alumni as CSV (PIN-protected) |
| `toast(msg)` | Show a brief notification |
| `dl(name, content, type)` | Trigger a file download |
| `genCode(batch, sec)` | Auto-generate student code |
| `loadFromFirestore()` | Fetch all data (incl. all 9 pending queues, 15 Firestore docs total) from Firestore on startup; falls back to localStorage |
| `compressImage(file, maxW, quality)` | Client-side canvas compress before storing photo as base64 (default 1200px, 0.72 quality); rejects the Promise with an error if the file is not a valid image (`img.onerror`) — caller in `saveEvent()` catches this and shows a toast |
| `changeFont(d)` | Increment/decrement font size (range 10–24px, default 18); writes to `--fs` CSS custom property and localStorage |
| `openLightbox(src)` | Open full-screen photo lightbox |
| `escapeHtml(s)` | HTML-encode a value for safe DOM injection — encodes `&`, `<`, `>`, `"`, `'`; called on every user-supplied field rendered into tables, cards, and rosters |
