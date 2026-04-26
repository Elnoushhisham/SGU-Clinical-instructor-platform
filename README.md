# Clinical Roster — Year 2 Spring 2026

A mobile-first Progressive Web App for clinical instructor assignment review and coordinator audit. Single-file HTML app; works on iPhone, Android, tablets, and desktop. No server, no install, no login required for v1 — just open `index.html` in a browser or save it to your phone's home screen.

What's in this folder

- `index.html` — the entire app (HTML + CSS + JavaScript + your pre-loaded data, ~2.7 MB; loads in <1 s on any modern phone)
- `seed-data.json` — your workbook parsed into the normalized schema. The same JSON is embedded in `index.html`; this copy is for inspection or backup
- `parse_workbook.py` — the Python script that produced `seed-data.json` (kept for reference)
- `README.md` — this document
- `DEPLOY.md` — step-by-step guide for getting the app online (GitHub Pages / SharePoint / Netlify)

How to run it

1. Double-click `index.html` to open in any modern browser (Safari, Chrome, Edge, Firefox).
2. To install on a phone, host the file somewhere reachable (GitHub Pages, Netlify, your school's web server, a Box public link), open it in mobile Safari/Chrome, then "Add to Home Screen".
3. Default coordinator passcode is `1234`. Change it under Coordinator → Settings.

The browser remembers your data in `localStorage`, so closing and reopening preserves state on the same device. Coordinators can re-import an updated `.xlsx` from the Import page or wire up live sync (next section).

## Live sync from OneDrive / SharePoint

Coordinator → **Import / Sync** → paste the share link to your master `.xlsx`. The app converts it to a direct-download URL automatically and fetches the file every time you open the app (and on demand via the **↻** button in the top bar).

**Setup (one-time, ~2 min):**
1. In OneDrive/SharePoint, open the workbook and click **Share → Anyone with the link → View**. Copy the link.
2. Paste it into the Import / Sync page.
3. Click **Save & sync now**. You'll see a green "Synced" badge with the timestamp.
4. Tick **Auto-sync each time the app opens** (default on) so coordinators and instructors always see the latest version.

**Behind the scenes:**
- SharePoint Online / OneDrive for Business links (`https://*.sharepoint.com/:x:/...`) get `?download=1` appended.
- OneDrive personal links (`https://1drv.ms/...` or `https://onedrive.live.com/...`) are routed through Microsoft's Shares API: `https://api.onedrive.com/v1.0/shares/u!<base64-url>/root/content`.
- Both endpoints stream the live `.xlsx` to the browser, where SheetJS parses it and the same importer used for the file-upload path produces the normalized JSON.

**If sync fails with a CORS error**, your tenant blocks cross-origin file downloads from outside its domain. The two practical workarounds:
1. Host the PWA on a SharePoint document library inside the same tenant — same-origin, no CORS issue.
2. Switch to a different storage layer with permissive CORS (a Box public link, a GitHub Pages-hosted copy, or your institutional web server). Paste that URL in the same field.

The **"Open download URL in tab (test)"** button is for the 5-minute compatibility check: paste your link, click it, and see whether OneDrive returns the file or an HTML page. If the file downloads in the new tab, the app's fetch will work too. If you see an HTML login page, the link permission needs to be loosened or the URL needs to be retrieved differently.

## Leave of absence requests

Instructors can request leave from inside the app. Coordinator → **Leave** holds the approval queue, the historical-leave importer, and an approved-leave heatmap.

### The approval rule

A request is **eligible for auto-approval** only if, for every day in the requested period, **fewer than 3 other instructors** are already on approved leave. If any day is at or above the cap, the request is **flagged** but still goes through to the coordinator — they make the final call. The cap value is `LEAVE_DAILY_LIMIT = 3` near the top of the script if you ever need to change it.

### Instructor flow (mobile)

1. Instructor → **Leave** tab.
2. Pick a leave type (Personal, Sick, Bereavement, Conference, Vacation, Maternity/Paternity, Other), a start date, an end date, and an optional reason.
3. As they pick dates, a live day-by-day table shows how many other instructors are already approved on each day, with a green OK / red Full badge.
4. Tap **Submit request**. The app stores the request locally and immediately opens a "Send to coordinator" panel with a pre-filled email (subject + body containing all the details and the eligibility check). The instructor types the coordinator's address (the app remembers it), taps **Open mail app**, and their phone's mail client takes over.
5. The request appears in the instructor's "My leave requests" list as **pending** until the coordinator records the decision.

### Coordinator flow

1. Coordinator → **Leave** tab. The tab itself shows a pending count badge in the nav.
2. Pending requests appear at the top, each with the day-by-day cap check inline. **Approve** or **Reject** with an optional note.
3. The **Add a request manually** card lets you log a request the instructor sent by email (pick instructor, type, dates) without round-tripping. Once added, it lands in the Pending queue and you decide there.
4. Below: an **approved-leave heatmap** for any month — cells in red are at or over the cap.
5. The **Approved & rejected history** section is collapsible.

### Importing already-approved leave

Coordinator → Leave → bottom of the page → **Import historical leave records**. Drop one or more `.xlsx` files. The importer auto-detects two layouts:

| Layout | Columns it looks for |
|---|---|
| **Long** (one row per leave) | `Name` (or `Instructor` / `Full name`), `Start Date` (or `From`), `End Date` (or `To` / `Until`), optional `Type`, optional `Reason`/`Note`, optional `Email` |
| **Daily** (one row per day) | `Name`, `Date` (a single date column), optional `Type`, optional `Reason`, optional `Email` |

If your file uses the daily layout, the importer groups consecutive same-type rows for the same instructor into a single range (e.g. three rows for Mar 30 / 31 / Apr 1 with type "Sick" become one Mar 30 → Apr 1 record).

Imported records land with `status=approved` and `source=imported:<filename>`. They count toward the daily cap immediately.

### Notifications — what's possible today, and the path to real push

The app is purely client-side, so out of the box "notification" means three things:

1. **In-app pending badge.** The coordinator's *Leave* tab shows the pending count in the navigation, and the dashboard has a **Pending leave** KPI card. Anyone opening the app sees the queue immediately.
2. **Mailto handoff.** The instructor's submit flow opens their mail client with the full request pre-filled. The coordinator gets a real email with all the details and the eligibility check — no "open the app to see it" required. This is what most users actually want.
3. **Cross-device sync.** The coordinator records the decision in their own copy of the app. To replicate the decision on the instructor's phone, just text/email them or rely on them re-opening the app the next time they want to check.

For **real push or two-way sync**, you need a tiny backend. Three options, smallest to largest:

- **Microsoft Power Automate flow + a "Leave Requests" sheet in your master `.xlsx`.** Set up a Forms trigger that emails you when an instructor submits, *and* appends a row to the workbook. The OneDrive sync in the app then pulls those rows into everyone's view automatically. Cost: free with M365. Effort: ~30 min one-time setup.
- **A free Cloudflare Worker** that exposes a `/leave` POST endpoint. The app POSTs each new request, the Worker emails the coordinator (via a free service like Resend or MailChannels) and stores rows in Cloudflare's KV store, which the app reads back on each open. Cost: $0/month. Effort: ~2 hours.
- **A SharePoint List + Power Automate.** Pure Microsoft stack. The app writes to the list via a public endpoint, Power Automate sends the email and updates approval status which the app reads back. Best for IT-controlled environments. Effort: ~2 hours with someone who knows Power Automate.

I haven't built these — they need decisions about your tenant policies — but the app is wired so adding any of them is mostly a small `submitToBackend(request)` function and a periodic refetch.

## Absence tracking

The April / May sheets capture absence as a cell value (`Absent`, `E.Absent`, `Absent all Day`, `E.Absent all Day`). The app counts these per instructor as **unique dates** — multiple absence cells on the same day count as one day, exactly as you asked.

Coordinator → **Attendance** tab shows a sortable table:
- Total absent days
- Unexcused (`Absent`, `Absent all Day`)
- Excused (`E.Absent`, `E.Absent all Day`)
- Recent dates inline + a click-through modal with the full list
- CSV export with one row per instructor and pipe-separated date lists for excused / unexcused

The instructor's own profile shows the same numbers for their personal record. The admin Dashboard surfaces a top-line **Absence-days** KPI (currently **71** across all 241 instructors; **Ahmed Alasaad Elamir** is the highest at 10 days, 8 unexcused).

---

## 1. Recommended database structure

Five tables, all flat, all keyed by stable IDs so re-imports merge cleanly. The app uses the same shape internally as JSON arrays.

### `instructors`
| field | type | notes |
|---|---|---|
| `instructor_id` | string PK | `inst_<8-char hash>` derived from email when present, else from normalized name |
| `full_name` | string | trimmed, single-spaced, titles stripped (`Dr.`, `Prof.`, etc.) |
| `email` | string | lower-cased, primary unique identifier when available |
| `department` | string | e.g. `Pathophysiology`, `Clinical Skills` |
| `department_slot` | string | e.g. `Pathology 3` — the labelled slot used in the original spreadsheet |
| `active_status` | enum | `active` \| `resigned` \| `pending` |
| `source_sheets` | string[] | which worksheets contributed this record — used for cross-sheet auditing |

### `master_schedule`
| field | type | notes |
|---|---|---|
| `activity_id` | string PK | `act_<hash>` from date+start+name+location |
| `course` | string | `PCM`, `BPM`, `CLSK`, `PHAR`, etc. |
| `activity_name` | string | e.g. `A: PCM1 FTCM SG 01 - HPWP` |
| `activity_type` | enum | `SG`, `SGP`, `ITI`, `Lecture`, `Review`, `BLS`, `IMCQ`, etc. |
| `date` | ISO date | `YYYY-MM-DD` |
| `start_time` / `end_time` | `HH:MM` | 24-hour |
| `location` | string | normalized room/venue |
| `discipline` | string | category code (`CLSK`, `PHAR`, …) |
| `description` | string | session topic |
| `required_number_of_instructors` | int | default 1 |
| `notes` | string | |
| `worksheet_source` | string | originating sheet name |

### `instructor_assignments`
| field | type | notes |
|---|---|---|
| `assignment_id` | string PK | |
| `activity_id` | FK → master_schedule | |
| `instructor_id` | FK → instructors | |
| `date`, `start_time`, `end_time`, `location` | denormalized for fast queries | |
| `role` | string | `Lead`, `Co-facilitator`, `SG Backup`, `Standby`, `Shadow`, `Off Duty` … (terms taken from your `Coding` sheet) |
| `worksheet_source` | string | |
| `notes` | string | |

### `review_sessions`
| field | type | notes |
|---|---|---|
| `review_id` | string PK | |
| `linked_activity_id` | FK → master_schedule (nullable) | populated by SG-code matching at import time |
| `course`, `module`, `discipline` | string | |
| `review_date`, `review_start_time`, `review_end_time`, `duration` | | |
| `review_location`, `zoom_link`, `meeting_id`, `passcode` | | |
| `sg_code` | string | e.g. `MNI SG1` |
| `notes` | string | |

### `review_attendance`
| field | type | notes |
|---|---|---|
| `review_id` | FK | |
| `instructor_id` | FK | |
| `attendance_required` | bool | derived: any instructor on the linked SG is required |
| `attendance_confirmed` | bool/null | for follow-up tracking after the fact |
| `source` | enum | `auto-derived` \| `manual` \| `imported` |
| `coordinator_note` | string | for overrides |

### Auxiliary
- `locations` (string[]) — derived
- `courses` (string[]) — derived
- `overrides` (`Map<issue_key, {resolved, note}>`) — coordinator-confirmed waivers, kept in client storage

A standard relational schema (Postgres / SQLite / Supabase) drops in 1:1 with the JSON. For a hosted v2, use UUIDs in place of the hash IDs and add `created_at` / `updated_at`.

---

## 2. Workflow for importing the Excel data

The pipeline runs in seven stages. The Python parser (`parse_workbook.py`) and the in-app importer (`parseWorkbook()` in `index.html`) implement the same logic; the in-app version uses [SheetJS](https://sheetjs.com).

```
┌─────────┐  ┌───────────┐  ┌────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐
│ .xlsx   │→ │ Sheet     │→ │ Per-sheet  │→ │ Cross-   │→ │ Conflict │→ │ Persist  │→ │ Render │
│ upload  │  │ classifier│  │ extractors │  │ linking  │  │ flagging │  │ to local │  │ views  │
└─────────┘  └───────────┘  └────────────┘  └──────────┘  └──────────┘  └──────────┘  └────────┘
```

1. **Upload** — coordinator picks an `.xlsx` file in the Import page. SheetJS reads it client-side; nothing leaves the device.
2. **Sheet classifier** — sheet names are matched case-insensitively against known patterns (`B-Line sessions`, `Review sessions Timetable`, `Clinical Instructor Name`, the four department sheets, `Deparmtment for the link`, `Contact Infromation`). Unknown sheets are skipped with a warning.
3. **Per-sheet extractors** — each known sheet has a tailored extractor that:
   - finds the header row by looking for required tokens (e.g. "subject", "start date" for B-Line; "first name", "last name" for the instructor list);
   - normalizes dates (Excel serials → ISO), times (`4:05pm` → `16:05`), and strings (trim + collapse whitespace);
   - drops sentinel rows (`Not Yet added/Resigned`, `#REF!`, blank).
4. **Cross-linking** — instructor records with the same email *or* the same bare name are merged; `source_sheets` is appended so duplicates remain auditable. Review sessions are linked to SG activities by matching `sg_code` against `activity_name` (e.g. `MI SG2` → `B: PCM1 FTCM SG 02`).
5. **Conflict flagging** — the conflict engine runs on the imported set and the results are cached on the dashboard.
6. **Persist** — payload goes to `localStorage` under key `clinical_roster_v1` (~250 KB typical; well under the 5 MB browser limit).
7. **Render** — UI re-routes to the dashboard.

To replace vs merge: the import preview offers two buttons. Replace = clean slate (use after a major schedule revision). Merge = upsert by primary key (use for incremental updates).

For Google Sheets workflow: **File → Download → Microsoft Excel (.xlsx)** then upload. v2 can call the Sheets API directly.

---

## 3. Cross-checking logic / rules

Each rule has a stable `issue_key` so the coordinator's "Mark as OK" overrides survive re-imports.

| # | Rule | Severity | issue_key prefix |
|---|------|----------|-----------------|
| R1 | Every activity in `master_schedule` should have ≥ `required_number_of_instructors` rows in `instructor_assignments` | Yellow | `unassigned\|<activity_id>` |
| R2 | No instructor is assigned to two activities whose time ranges overlap on the same day | Red | `conflict\|<inst>\|<date>\|<a>\|<b>` |
| R3 | Names are trimmed, single-spaced, titles stripped before equality checks | (data hygiene) | — |
| R4 | Every `instructor_assignments.activity_id` must exist in `master_schedule` (orphan check) | Red | `orphan\|<assignment_id>` |
| R5 | Every `master_schedule` row should have at least one assignment (= R1) | Yellow | — |
| R6 | If an SG / SGP activity has a matching `review_sessions.linked_activity_id`, mark every assigned instructor as `attendance_required = true` | (auto rule) | — |
| R7 | Same as R6 — the implementation | (auto rule) | — |
| R8 | The instructor-side review view shows date, time, location, activity, and required attendees | (UX rule) | — |
| R9 | SG-assigned instructor missing from review attendance | Yellow | `sgmissrev\|<inst>\|<review>` |
| R10 | Review attendee not on the SG roster | Yellow | `revnotsg\|<inst>\|<review>` |
| R11 | Review session with no `linked_activity_id` | Yellow | `revnosg\|<review>` |
| R12 | SG/SGP activity with no linked review | Yellow | `sgnorev\|<activity>` |
| R13 | Same activity name appearing on different dates across sheets | Yellow | (informational) |
| R14 | Coordinator can mark any flagged issue resolved with a note; resolution is keyed and persisted | (UX rule) | overrides map |

Plus three operational rules:
- **R15** — assignments outside 8 AM – 5 PM raise a yellow flag (`hours\|<inst>\|<activity>`).
- **R16** — possible duplicate instructor names (same bare name across multiple records).
- **R17** — orphan reviews (review session whose `linked_activity_id` no longer resolves).

---

## 4. Mobile-friendly interface plan

The app is built mobile-first; layouts up-scale via three breakpoints.

```
< 600 px  : single column, sticky top bar, sticky tabs, bottom-pinned actions
600–960 px: two-column KPI grids, calendar collapses to month view
≥ 960 px  : wide table views, side-by-side detail/list (centered max-width 980px)
```

Conventions:
- Touch targets ≥ 38 × 38 px. Form fields are `0.7rem` padding for fat-finger friendliness.
- High-contrast palette with WCAG AA contrast in both light and dark mode (auto-follow `prefers-color-scheme`).
- Status colors: green (no issue) · yellow (needs review) · red (conflict / missing).
- Two view modes everywhere relevant: **Table** for density, **Calendar** for a glance.
- Search lives in the top bar; one tap from any screen.
- Bottom-of-screen action bar respects iOS safe area (`env(safe-area-inset-bottom)`).
- The icon, manifest, and Apple meta tags make "Add to Home Screen" work on iOS without a service worker; on Android a minimal in-page service worker upgrades to true offline.

---

## 5. Admin dashboard structure

```
Coordinator
├── Dashboard           ← KPIs + audit summary scorecard, every issue with a count and a link
├── Schedule            ← Master schedule, table or calendar, filterable by instructor/course/location/type
│   └── Activity drawer ← Tap a row to see assigned instructors and the linked review session
├── Conflicts           ← All open issues grouped by rule, each with "Mark as OK + note"
├── Data cleaning       ← Duplicate-name groups, name-trim button, mismatch detector,
│                          export of the canonical instructor list
├── Import              ← Drop an .xlsx, see counts preview, choose Replace or Merge
├── Export              ← One-click CSV per table, plus a single "audit PDF" via browser print
│                          plus per-instructor schedule download
└── Settings            ← Passcode, JSON backup, factory reset
```

KPIs surfaced on the dashboard: instructor count, activity count, review count, assignment count, **time conflicts**, **unassigned activities**, **outside-hours assignments**, **possible duplicate names**.

---

## 6. Instructor dashboard structure

```
Instructor
├── Home                ← Greeting, KPIs, "Next up" card, upcoming review preview
├── My assignments      ← Filter by day-of-week / month / course / type;
│                          item cards with date, time, location, activity, role, "Add to calendar"
├── Review sessions     ← Required vs Open badges; Zoom links open in browser
├── Calendar            ← Month grid with assignments (blue), reviews (yellow), conflicts (red)
└── Profile             ← Email, department, slot, source sheets, "Switch instructor"
```

If no assignments are mapped to the instructor's name yet, the **My assignments** view falls back to "all sessions for your department" with a banner so the page is never blank during onboarding.

---

## 7. Suggested technology options

| Concern | v1 (this build) | v2 if you outgrow it |
|---|---|---|
| Hosting | Static HTML, drop on GitHub Pages / Netlify / Box public link | Same |
| Storage | Browser `localStorage` | Postgres on Supabase / Firebase Firestore |
| Auth | Coordinator passcode, no instructor auth | Magic-link email (Supabase Auth) or institution SSO |
| Excel import | SheetJS (client-side) | Server-side `openpyxl` + Cloud Storage |
| PDF export | Browser print of an audit page | `weasyprint` server-side, or `jsPDF` if you want fully client |
| Offline | Minimal in-page Service Worker (Android), Apple "Add to Home Screen" (iOS) | `vite-plugin-pwa` with proper precaching |
| Calendar | `.ics` download | OAuth + Google/Outlook write API |
| Notifications | None | Web Push + email digest job |
| Stack rebuild path | n/a | SvelteKit or Next.js + tRPC + Postgres + Tailwind |

Why no framework in v1: a single-file HTML app loads in <100 ms on a phone, has zero build step, and can be emailed as an attachment. The cost is hand-rolled DOM, but the surface area is small enough (~1500 lines) that it stays maintainable.

---

## 8. Step-by-step development plan

1. ✅ **Schema + parser** — `parse_workbook.py` ingests the workbook into normalized JSON. *Done.*
2. ✅ **Single-file PWA shell** — routing, mobile layout, theming, modal/toast primitives. *Done.*
3. ✅ **Instructor views** — picker, dashboard, assignments, reviews, calendar, profile. *Done.*
4. ✅ **Coordinator views** — dashboard, schedule, conflicts, cleaning, import, export, settings. *Done.*
5. ✅ **Conflict engine** — R1, R2, R6–R12, R15–R16. *Done.*
6. ✅ **Excel re-import inside the app** — SheetJS dynamic load + replace/merge. *Done.*
7. ✅ **Exports** — CSV per table, per-instructor schedule, `.ics` calendar, print-PDF audit. *Done.*
8. ⏭ **Manual assignment editor** — add/edit/delete instructor assignments inside the app for activities the spreadsheet didn't carry assignments for.
9. ⏭ **Review attendance manual entry** — coordinator can confirm attendance directly.
10. ⏭ **Activity name fuzzy matcher** — for R13 mismatches, propose canonical names.
11. ⏭ **Magic-link auth** — once you host on a real backend.
12. ⏭ **Cloud sync** — replace `localStorage` with a hosted DB; keep the same JSON shape.
13. ⏭ **Calendar OAuth integration** — push to instructor's Google/Outlook calendar instead of `.ics`.
14. ⏭ **Push notifications** — "you have an assignment in 30 minutes."

Steps 1–7 ship as v1. Steps 8–10 are the obvious v1.1. Steps 11–14 are v2.

---

## 9. Pseudocode for the error-checking logic

```
function detectConflicts(assignments):
    by_instructor = group(assignments, key=instructor_id)
    out = []
    for inst, items in by_instructor:
        items = sort(items, by=(date, start_time))
        for i in 0..len(items)-1:
            for j in i+1..len(items)-1:
                if items[j].date != items[i].date: break          # sorted, so we can stop
                if overlaps(items[i], items[j]):
                    out.push({ instructor_id: inst, date: items[i].date,
                               a: items[i].activity_id, b: items[j].activity_id })
    return out filter not_overridden

function overlaps(a, b):
    return min(a.start, a.end) < max(b.start, b.end)
       and min(b.start, b.end) < max(a.start, a.end)
       # equivalently: a.start < b.end AND b.start < a.end

function detectUnassigned(master_schedule, assignments):
    assigned = set(a.activity_id for a in assignments)
    return [act for act in master_schedule if act.activity_id not in assigned]

function detectOutsideHours(assignments, lower=8*60, upper=17*60):
    return [a for a in assignments
            if min_of(a.start_time) < lower or min_of(a.end_time) > upper]

function detectSgWithoutReview(master_schedule, review_sessions):
    linked = set(r.linked_activity_id for r in review_sessions if r.linked_activity_id)
    return [act for act in master_schedule
            if act.activity_type in {'SG', 'SGP'} and act.activity_id not in linked]

function detectSgMissingReview(review_sessions, assignments, review_attendance):
    out = []
    for r in review_sessions:
        if not r.linked_activity_id: continue
        sg_instructors = { a.instructor_id for a in assignments if a.activity_id == r.linked_activity_id }
        present        = { x.instructor_id for x in review_attendance if x.review_id == r.review_id }
        for inst in sg_instructors - present:
            out.push({ instructor_id: inst, review_id: r.review_id, activity_id: r.linked_activity_id })
    return out

function detectDuplicateNames(instructors):
    groups = group(instructors, key=lambda i: bare(i.full_name))
    return [g for g in groups.values() if len(g) > 1]

function bare(name):
    return collapse_whitespace(name).strip().lower()
                                    .replace(/^(dr\.?|mr\.?|mrs\.?|ms\.?|prof(essor)?\.?)\s+/, '')

function timeToMin(s):
    match HH:MM in s; return HH*60 + MM
```

For the **R6/R7 auto-rule** (review attendance derivation), in Excel/Sheets terms:

```
=IF(
   IFERROR(MATCH(SG_CODE, ACTIVITIES_SG_COLUMN, 0), 0) > 0,   /* this review has a linked SG */
   IF(IFERROR(MATCH(INSTRUCTOR_ID, SG_INSTRUCTORS_FOR_THAT_SG, 0), 0) > 0,
      "Required",
      "Open"),
   "(no linked SG)"
)
```

And for a quick **double-booking flag** as an Excel formula in an `Assignments` sheet (column-by-column compare):

```
=IF(SUMPRODUCT(
        ($A2 = $A$2:$A$1000)                          /* same instructor       */
      * (B2 = B$2:B$1000)                             /* same date             */
      * (D2 > C$2:C$1000) * (C2 < D$2:D$1000)         /* time overlap          */
      * (ROW($A$2:$A$1000) <> ROW())                  /* exclude self          */
   ) > 0, "CONFLICT", "")
```

Where columns are A=instructor, B=date, C=start, D=end.

---

## 10. Assumptions and questions before development

Assumptions baked into v1 (worth confirming):

- Names are matched case-insensitively, whitespace-collapsed, and with `Dr.`/`Prof.`/`Mr.`/`Ms.`/`Mrs.` titles stripped. Two records that differ only in titles or spacing are treated as the same person.
- Email is the most reliable instructor identifier when present. Otherwise the bare name is used.
- "Working hours" for the outside-8-to-5 flag are local times as stored in the spreadsheet — no timezone conversion. (Grenada local time everywhere is the safe assumption for SGU; let me know if any of these are remote and zone-aware.)
- The April / May "month" sheets are the authoritative source for per-session instructor assignments. They use a wide layout: row 2 = date (sparse, forward-filled), row 3 = course, row 4 = session name (forward-filled across cohort columns), row 5 = cohort A/B/C/D, row 7 = venue (forward-filled), row 10 = start time, and from row 11 down each row is one instructor with a non-empty cell at every (column = session/cohort) where they are assigned. The cell value is the instructor's role at that session — `Table 13`, `ITI Table 04`, `Backup`, `Standby`, `Absent`, `E.Absent`, `Shadow`, etc.
- Assignment cells whose role is `Absent`, `E.Absent`, `Off`, `Off Duty`, `Tec. Failure`, `Internet Issue`, `Other`, `Test`, or `Shadow` are treated as recorded-but-not-occupying — they're imported (so you can audit them) but the conflict engine and the outside-hours check skip them. Anything that names a table or otherwise indicates active duty (`Table NN`, `ITI Table NN`, `Backup`, `Backup SIM`, `Coordinator ITI`, `BLS Facilitator`, `BLS Candidate`, `Standby`, `Half SG`, etc.) is treated as a real assignment. If you want different roles in the "non-occupying" bucket, edit `NON_ACTIVE_ROLES` at the top of the conflict-engine section.
- Cohort labels (A / B / C / D) on the same date can run either staggered (A 8:00, B 10:00, C 8:00, D 10:00) or all-parallel. The importer treats each cohort as a separate activity, so an instructor listed at `Table 12` in two cohorts that share a start time will surface as a conflict — which is correct: those are parallel rooms running simultaneously.
- The B-Line sessions sheet is loaded as the secondary master schedule (mostly historical PCM1 sessions). PCM2-Main, PCM2-Alternate, and BPM3 are not yet flattened — they're the planning-grid view of the same data the April/May sheets express. v1.1 can ingest them too if you want a fuller multi-month view.
- Review-to-SG linking is by SG code substring match (e.g. `MNI SG1` vs `PCM1 FTCM SG 01`). This is approximate and produces some unlinked reviews you'll see in the audit. The `Mark as OK` overrides are designed for this exact case.
- "Required attendance" for a review is auto-derived from "are you on the linked SG's roster". Manual overrides can either add or remove an instructor.
- All data is local to the device. No sync, no backups, no multi-user editing in v1. If two coordinators each edit on different phones, last-import-wins.
- The four department sheets (`Pathophysiology`, `Pathology`, `Clinical Skills`, `Micro Pharm`) define department membership only — they do not specify which sessions an instructor will run. Per-session assignment data is what's missing to fully populate `instructor_assignments`. Today the v1 import yields 0 assignments unless the workbook adds them.

Open questions to confirm:

1. **Per-session assignment source.** ✅ Confirmed: the April / May "month" sheets are the authoritative instructor↔session matrix. Row 2 = date, row 4 = session, row 5 = cohort, row 7 = venue, row 10 = time, rows 11+ = one instructor per row with cell = role/table. The importer now flattens this into 8,300+ assignment records.
2. **"Activity codes."** Should there be a single canonical activity code (e.g. `S26-CLSK-SG-01-A`) that flows through every sheet, or is the human-readable subject the source of truth?
3. **Multiple cohorts (A/B/C/D).** Are A/B/C/D separate runs of the same activity (4 distinct activity_ids) or one activity with four instructor slots? The current import treats them as separate activities; flag if you'd prefer the second model.
4. **Required instructor count per activity.** Default is 1. Are there sessions that need 2+ (BLS facilitators, SIM lab pairings)? If so, a column on the master schedule sheet would let the unassigned-detector be more precise.
5. **Outside-hours window.** v1 hardcodes 8 AM – 5 PM. Should evening/weekend sessions (BLS, SIMLAB, Zoom reviews) be exempt? A per-activity "outside hours OK" flag is easy to add.
6. **"Pre-clerkship vs other terms."** This file is "Year 2 Spring 2026". Will the same app load Fall 2026 and prior terms, or do we keep one app per term?
7. **Personally identifiable information.** OK to keep instructor emails in plain text in the offline app? If not, we'd switch to displaying initials only and matching by ID.
8. **Hosting.** GitHub Pages? An SGU-controlled URL? Box public link? This determines the "Add to Home Screen" UX (PWAs work best from HTTPS).
9. **Calendar integration.** `.ics` download is in. Should we also wire up Google/Outlook OAuth in v2? That requires backend, so it's worth knowing now.
10. **Resigned / inactive instructors.** Should they vanish from name pickers or stay searchable for historical records?

---

## File map (cheatsheet)

```
clinical-roster/
├── index.html         ← The PWA. Open it. That's it.
├── seed-data.json     ← Pre-parsed copy of your workbook (also embedded in the HTML)
├── parse_workbook.py  ← The Python that produced seed-data.json
└── README.md          ← This file
```

If you hit a bug, the keyboard shortcut to open browser dev tools is **F12** (desktop) — the console will surface any error from the conflict engine or import path. The same data shape is dumped from Coordinator → Settings → Download backup (JSON), which is the easiest way to share state when reporting an issue.
