# Putting the Clinical Roster app online

You have three realistic paths. Pick by what you care about most: speed, OneDrive sync compatibility, or staying inside the SGU tenant.

| Goal | Pick |
|---|---|
| Fastest demo to test on a phone today | **GitHub Pages** (15 min) |
| Best OneDrive sync, lives inside SGU's Microsoft 365 tenant | **SharePoint** (~30 min, needs IT) |
| No technical setup at all, just drag a folder | **Netlify Drop** (5 min) |

The same files work for all three. You're hosting `index.html`, `seed-data.json`, and `README.md` (plus `parse_workbook.py` if you want to keep it visible).

All three give you HTTPS — required for "Add to Home Screen" to work on iPhone and Android.

---

## Option 1 — GitHub Pages (recommended for the pilot)

**You'll get a URL like** `https://<your-username>.github.io/clinical-roster/`

**Cost:** free for public repos, free for private repos on a paid plan.

**Steps:**

1. Make a GitHub account if you don't have one (`github.com/signup`).
2. Click **New repository** → name it `clinical-roster` → **Public** (or Private if you have GitHub Pro) → **Create**.
3. On the repo's home page, click **Add file → Upload files**. Drag the three files from this folder (`index.html`, `seed-data.json`, `README.md`) into the upload area. Scroll down, click **Commit changes**.
4. Click **Settings** (the repo's settings, not your account's) → **Pages** in the left nav.
5. Under **Source**, choose **Deploy from a branch**. Branch = `main`, Folder = `/ (root)`. Click **Save**.
6. Wait ~1 minute. Refresh the Pages settings page; the URL appears at the top: `https://<your-username>.github.io/clinical-roster/`.
7. Open that URL on your phone. Add to Home Screen.

**To update the data:** edit `index.html` (or upload a new one), GitHub Pages re-deploys in ~1 minute. Even simpler — set up the OneDrive sync inside the app, then you only push code changes to GitHub when you change the app itself.

**Privacy note:** the public repo means anyone with the URL can see the seed-data. If your seed contains real instructor emails, use a **Private** repo (Pro plan, $4/month) or move to Option 2 / 3.

---

## Option 2 — SharePoint inside SGU's tenant (recommended once IT signs off)

**You'll get a URL like** `https://sgu.sharepoint.com/sites/<sitename>/SitePages/Roster.aspx`

**Why this is the "right" answer for OneDrive sync:** the PWA and the .xlsx live on the same Microsoft 365 tenant, so the browser sees them as same-origin. CORS issues vanish.

**You'll need:** site owner permission on a SharePoint site, or someone in IT to create one for you (a Communication Site or Team Site both work).

**Steps:**

1. Ask IT to create a SharePoint site (or use one you already own). Suggested name: "Year 2 Clinical Roster".
2. In that site, go to **Site contents → Documents** (or any document library).
3. Click **Upload → Files** and pick `index.html` and `seed-data.json` from this folder.
4. The trick: you want the HTML to *render*, not download. Two ways:
   - **a) Direct file URL** — Right-click `index.html` in the library → **Copy link** → choose **Anyone with the link can view**. The link will look like `https://sgu.sharepoint.com/:b:/r/sites/...`. Append `&web=1` if it doesn't open in the browser.
   - **b) SharePoint Page (cleaner)** — Create a new SharePoint Page. Add an **Embed** web part. Paste this:
     ```html
     <iframe src="<the index.html share link from step a>" style="width:100%;height:100vh;border:0"></iframe>
     ```
     Save the page. Now the URL of the page (`.../SitePages/Roster.aspx`) is what you give to instructors.
5. Upload your master `.xlsx` to the **same** site/library. Right-click → Copy link → **Anyone with the link can view**.
6. Open the app, go to Coordinator → Import / Sync, paste the .xlsx link, click Save & sync now. Because both files are on `sgu.sharepoint.com`, no CORS error.

**Heads-up:** SharePoint will rewrite some asset paths. The PWA is one self-contained file with all CSS/JS inline so it should serve cleanly, but the first time you do this, click around all the views to confirm everything loads.

**If your IT department blocks "anyone with the link" outside the tenant:** that's fine — share with **People in SGU** instead. Instructors will need to sign in once on first open, then it's invisible after that.

---

## Option 3 — Netlify Drop (fastest if you don't want a GitHub account)

**You'll get a URL like** `https://amazing-curie-1a2b3c.netlify.app/`

**Cost:** free.

**Steps:**

1. Open `app.netlify.com/drop` in your browser. (Make a free account first if needed — sign in with email is fine.)
2. Drag the entire `clinical-roster` folder onto the drop zone.
3. Wait 30 seconds. Netlify gives you the URL.
4. Optional: in Netlify's dashboard, **Site settings → Change site name** to something memorable like `sgu-roster`. URL becomes `sgu-roster.netlify.app`.

**To update the data:** drag a new copy of the folder onto the drop zone — Netlify replaces the site.

**Why this isn't the recommended pick:** zero version history. If you make a mistake, the previous version is gone. GitHub Pages keeps every change.

---

## After hosting — install on phones

This is the same on every option above.

**iPhone (Safari):**
1. Open the URL in Safari (not Chrome — iOS only allows Safari to install web apps).
2. Tap the **Share** icon (the square with the up-arrow).
3. Scroll down → **Add to Home Screen** → **Add**.

The app icon appears with the "CR" badge. Tapping it opens the app fullscreen, no browser chrome, like a real iPhone app.

**Android (Chrome):**
1. Open the URL in Chrome.
2. Tap the menu (⋮) in the top-right.
3. Tap **Add to Home Screen** (or **Install app** on newer Chrome).

Same result — fullscreen icon launch.

**Desktop:** in Chrome / Edge there's an install icon in the address bar (a screen with a down-arrow). Click it. The app gets its own dock/Start-menu entry.

---

## Sharing the right link with the right people

**Coordinator URL:** `https://<your-host>/index.html` — this is the same URL for everyone. The coordinator role is gated by the passcode you set in Settings (default `1234`, change it!).

**Instructor URL:** same as the coordinator URL. They pick their name on the welcome screen. The device remembers it.

**Pre-filled instructor link (if you want to email per-instructor):** not in v1 — every device has its own selection. v1.1 can add this with a query parameter if you ask for it.

---

## Setting up OneDrive sync after hosting

Once your hosted PWA is reachable at a URL:

1. Coordinator → Sign in with passcode → **Import / Sync**.
2. In OneDrive or SharePoint, open the workbook → **Share** → **Anyone with the link can view** → Copy link.
3. Paste the link into the **Sync from OneDrive / SharePoint** field.
4. Click **Open download URL in tab (test)** — a new tab should pop and either start downloading the .xlsx or open it in Excel Online. If it asks you to sign in or shows an error, the share link permissions need adjusting (try "People in your organization" or loosen the link further).
5. Once the test tab works, click **Save & sync now**. You should see a green "Synced" badge with the timestamp and the data refreshes.
6. Tick **Auto-sync each time the app opens** so coordinators and instructors always see the latest version (this is on by default).

After this, every edit you make in the Excel file appears in everyone's app on their next refresh. The **↻** button in the top bar forces a refresh on demand.

---

## Troubleshooting

**"Sync failed: Network/CORS error"**
The hosting domain and the OneDrive domain are different. Either:
- Move to **Option 2** (host on the same SharePoint tenant), or
- Switch the data file to a Box public link / GitHub-hosted .xlsx — both have CORS-permissive endpoints.

**"Downloaded file looks too small"**
The share link returned an HTML login page instead of the .xlsx. Loosen the link permissions (anyone with the link, view) or use a different storage layer.

**"Could not parse as .xlsx"**
The fetched bytes weren't a real Excel file. Open the share URL in a fresh browser tab — if it doesn't download a file there either, the link is broken.

**App opens but shows old data**
You hit cached data. Open Coordinator → Settings → **Reset to seed data** to clear local state, then re-sync. Or in the browser dev tools: Application → Storage → Clear site data.

**Add to Home Screen is greyed out**
You're not on HTTPS. All three hosting options above give you HTTPS by default. If you opened the file via `file://` on your phone, that won't install — host it.

---

## What I'd actually do, in order

1. Today (15 min): push to GitHub Pages, share the URL with two or three instructors as a smoke test, install it on your own phone.
2. This week (~30 min, with IT): set up the SharePoint version, upload the master `.xlsx` next to it, configure OneDrive sync. That's the production setup.
3. Once instructors have it: change the coordinator passcode from `1234` to something real; sign yourself out of coordinator mode on shared devices.
