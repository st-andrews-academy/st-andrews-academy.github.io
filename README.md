# St. Andrew's Academy — Alumni Portal


**School:** St. Andrew's Academy, Candaba, Pampanga, Philippines  
**Established:** 1946  
**Portal Version:** 1.2 — May 2026  
**Live URL:** https://st-andrews-academy.github.io

---

## 📋 Features

### 7 Dashboard Tabs
| Tab | Description |
|-----|-------------|
| 📊 **Overview** | Alumni summary, statistics, section breakdown, 2026 Reunion contributions & teacher attendance |
| 📜 **History** | School history timeline, photo gallery, 1986 world events |
| 🎓 **Graduation** | Batch year selector, class portraits (Batch 1986 Sections A–G), generated class rosters |
| 👥 **Alumni DB** | Full alumni database — filter, sort, add, edit, delete, export, backup |
| 👩‍🏫 **Faculty** | Faculty directory — advisers, 2026 reunion attendance, add/edit/delete records |
| 📸 **Memory Lane** | Event albums with photos — chronological view, add events |
| ⚙️ **Settings** | Backup/restore database, export CSV, Memory Lane backup, change Admin PIN, pending registrations |

### Key Capabilities
- 📝 **Alumni Registration** button always visible on top nav — public self-registration
- ⏳ **Pending Approvals** — Add/Edit submissions go to a pending queue; admin reviews and approves (PIN-protected)
- 🔐 **PIN Protection** on admin actions (Approve/Reject, Delete, Restore, Backup/Export, Change PIN)
- 📥 **Export to CSV** — Excel-compatible export of full alumni database
- 💾 **JSON Backup & Restore** — for alumni, faculty, events and settings
- 🔎 **Filter & Sort** on all database columns
- 🖼️ **Photo Lightbox** — click any photo to enlarge
- 📱 **Mobile-friendly** — responsive design for phones and tablets
- 🔠 **Font size controls** (A+ / A−) — size 10 to 24, default 18
- 🛡️ **XSS Protection** — all dynamic HTML output is HTML-escaped via `escapeHtml()` before rendering
- 🔒 **Content Security Policy** — CSP meta tag restricts script/connect origins to Firebase and Google Fonts

---

## 🗃️ Database Contents (Seed Data)

| Dataset | Records |
|---------|---------|
| Alumni (Batch 1986) | 321 graduates across 7 sections |
| Faculty | 15 records (7 Batch 1986 advisers + 8 reunion attendees) |
| History Photos | 17 embedded images |
| Class Photos | 7 (Sections A–G, Batch 1986) |
| 2026 Reunion Contributions | 39 listed, ₱225,200 total (98 contributors) |
| 2026 Reunion Teachers | 13 (10 attended, 3 could not attend) |

---

## 🔐 Admin PIN

Default PIN is pre-set. Change it anytime via **Settings → Change Admin PIN**.  
Protected actions: Approve/Reject pending submissions, Delete, Restore, Backup/Export, Change PIN.  
Adding and editing records is open to anyone — submissions go to a pending queue for admin review.

**Brute-force lockout:** 5 consecutive wrong PIN attempts triggers a 60-second lockout. The lockout state is stored in `sessionStorage` so it survives a page refresh within the same browser session.

---

## 💾 Data Storage & Backup

This portal uses **Firebase Firestore** as its live database, with browser localStorage as a read cache only. Every page load fetches fresh data from Firestore and overwrites localStorage — Firestore is the sole source of truth. If Firebase is unreachable on load, a persistent red offline banner appears and all writes are blocked until the user reloads successfully.

| Storage | How it works |
|---------|-------------|
| **Firestore (primary)** | All edits sync to the cloud automatically — shared across all users |
| **Browser localStorage** | Local read cache only; refreshed from Firestore on every page load |
| **JSON Backup** | Download via Settings or Alumni DB — full offline snapshot |
| **CSV Export** | Excel-compatible export of alumni records |
| **Memory Lane Backup** | Downloads all events and photos as JSON |

> **Note:** Because data lives in Firestore, editing on one device is immediately reflected for everyone. You no longer need to re-upload `index.html` to share data changes — only re-upload when you need to update the app itself (new features, seed data for fresh installs, etc.).

> **Firebase Rules:** The Firestore security rules must allow `read: if true` for all portal documents (including `portal/config` and all 9 pending queue docs). The app reads all 15 Firestore documents in a single batch on startup — if any one read is denied, the entire load fails and the offline banner appears.

---

## 🌐 Hosting (GitHub Pages)

Already live at: **https://st-andrews-academy.github.io**

To update the portal:
1. Go to https://github.com/st-andrews-academy/st-andrews-academy.github.io
2. Click **Add file → Upload files**
3. Upload new `index.html` (replaces old one)
4. Click **Commit changes**
5. Live within 2–5 minutes

---

## 📁 Repository Files

| File | Description |
|------|-------------|
| `index.html` | Complete self-contained portal — all data, photos, styles and scripts embedded |
| `SAA_Architecture_Diagram.html` | Standalone architecture diagram — also deployed to GitHub Pages |
| `README.md` | This documentation file |

---

## 📬 Contact

**Email:** st_andrews_academy@hotmail.com  
**School:** St. Andrew's Academy, Candaba, Pampanga, Philippines  
**Batch 1986 — 40th Reunion:** April 11, 2026  
**Theme:** *"Andrewlite Ku, Pagmaragul Ku"* (I am an Andrewite, I am proud)
