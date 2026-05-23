# St. Andrew's Academy — Alumni Portal


**School:** St. Andrew's Academy, Candaba, Pampanga, Philippines  
**Established:** 1946  
**Portal Version:** 1.1 — May 2026  
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
- ⏳ **Pending Approvals** — registrations queue in Settings for admin review (PIN-protected)
- 🔐 **PIN Protection** on all write actions (Add, Edit, Delete, Restore, Approve/Reject)
- 📥 **Export to CSV** — Excel-compatible export of full alumni database
- 💾 **JSON Backup & Restore** — for alumni, faculty, events and settings
- 🔎 **Filter & Sort** on all database columns
- 🖼️ **Photo Lightbox** — click any photo to enlarge
- 📱 **Mobile-friendly** — responsive design for phones and tablets
- 🔠 **Font size controls** (A+ / A−) — size 10 to 24, default 15

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
Protected actions: Add, Edit, Delete, Restore, Import, Approve/Reject registrations.

---

## 💾 Data Storage & Backup

This portal uses **Firebase Firestore** as its live database, with browser localStorage as a fallback cache. All changes made by any user are saved to Firestore in real time and visible to all other users on their next page load.

| Storage | How it works |
|---------|-------------|
| **Firestore (primary)** | All edits sync to the cloud automatically — shared across all users |
| **Browser localStorage** | Local cache; used as fallback if Firestore is unreachable |
| **JSON Backup** | Download via Settings or Alumni DB — full offline snapshot |
| **CSV Export** | Excel-compatible export of alumni records |
| **Memory Lane Backup** | Downloads all events and photos as JSON |

> **Note:** Because data lives in Firestore, editing on one device is immediately reflected for everyone. You no longer need to re-upload `index.html` to share data changes — only re-upload when you need to update the app itself (new features, seed data for fresh installs, etc.).

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
| `README.md` | This documentation file |

---

## 📬 Contact

**Email:** st_andrews_academy@hotmail.com  
**School:** St. Andrew's Academy, Candaba, Pampanga, Philippines  
**Batch 1986 — 40th Reunion:** April 11, 2026  
**Theme:** *"Andrewlite Ku, Pagmaragul Ku"* (I am an Andrewite, I am proud)
