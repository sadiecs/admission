# Setup Guide

This guide takes you from a blank Google account to a fully working, deployed admission system. It assumes no prior Apps Script experience.

---

## 0. What you'll need

- A Google account (the "owner" account — it receives submission notifications by default, and its quota/limits apply to Sheets, Drive and email sending).
- About 20–30 minutes for the first-time setup.

---

## 1. Create the Google Sheet

1. Go to [sheets.google.com](https://sheets.google.com) and create a new blank spreadsheet.
2. Name it something recognizable, e.g. `UMV Chochahi Chhapra — Admissions`.
3. **Leave it empty** — `Code.gs` creates the `data`, `admins` and `auditLog` tabs automatically the first time it runs.
4. Copy the **Sheet ID** from the URL:
   ```
   https://docs.google.com/spreadsheets/d/  1AbCdEfGhIjKlMnOpQrStUvWxYz...  /edit
                                             └──────────── this part ────────────┘
   ```

---

## 2. Create the Google Drive folder

1. Go to [drive.google.com](https://drive.google.com) and create a folder, e.g. `UMV Admission Files`.
2. Open it and copy its **Folder ID** from the URL:
   ```
   https://drive.google.com/drive/folders/  1XyZ...
                                             └── this part ──┘
   ```
3. This folder holds every applicant's photo, signature and supporting documents (Aadhaar front/back, TC, Matric certificate). **These files stay private** — the script never sets public sharing on them, and you don't need to touch sharing settings. Access is only brokered through the deployed script (see [ARCHITECTURE.md](./ARCHITECTURE.md#file-storage--access-model)).

---

## 3. Create the Apps Script project

1. From inside the Sheet, go to **Extensions → Apps Script**.
2. Delete any placeholder code.
3. Copy the entire contents of `Code.gs` into the editor.
4. Save (`Ctrl/Cmd + S`).
5. Rename the project to something like `UMV Admissions Backend`.

---

## 4. Set Script Properties

1. In the Apps Script editor open **Project Settings** (gear icon).
2. Under **Script Properties** → **Add script property**, add:

| Property | Value | Required? |
|---|---|---|
| `SHEET_ID` | Sheet ID from step 1 | **Yes** |
| `DRIVE_FOLDER_ID` | Folder ID from step 2 | **Yes** |
| `ADMIN_PASSWORD` | A password of your choice, e.g. `ChangeMe123!` | Yes, unless you set up the `admins` tab ([step 9](#9-optional-set-up-per-user-admin-accounts)) |
| `OWNER_NOTIFY_EMAIL` | Address for owner notifications | No — defaults to the deploying Google account |

> `ADMIN_TOKEN_SECRET` is **not** something you set — the script generates it the first time an admin logs in.

3. Save.

---

## 5. Deploy as a Web App

1. Click **Deploy → New deployment**.
2. Click the gear icon next to "Select type" → **Web app**.
3. Fill in:
   - **Description:** anything, e.g. `v1`
   - **Execute as:** **Me** — lets the script read/write the Sheet and Drive without visitors needing a Google account.
   - **Who has access:** **Anyone** — the public form and dashboard must reach it without a Google login.
4. Click **Deploy**.
5. On first deploy Google asks you to **authorize** the app — click through, choose your account, "Advanced" → "Go to [project name] (unsafe)" if warned (expected for a private, unverified script you wrote yourself), and **Allow** (Sheets, Drive, Gmail).
6. Copy the **Web app URL**:
   ```
   https://script.google.com/macros/s/AKfycb.../exec
   ```

> ⚠️ **The live URL serves a pinned version.** After editing `Code.gs` you must publish a new version — see [Updating the backend later](#updating-the-backend-later).

---

## 6. Point the frontend files at your deployment

1. In `index.html`, find `CONFIG.WEB_APP_URL` and replace the URL with the one from step 5.
2. Do the same in `admin.html` (it has its own `CONFIG.WEB_APP_URL`).
3. Save both.

---

## 7. Host (or open) the frontend files

- **Just testing:** double-click `index.html` / `admin.html`; `fetch` calls to the Apps Script URL work without a local server.
- **Real deployment:** upload `index.html` (and `admin.html`) to any static host — GitHub Pages, Netlify, Google Sites, or your school's hosting.
- **Keep the dashboard low-profile:** `admin.html` has its own login; hosting it at a less obvious path adds a thin extra layer, but the login is what protects it — don't rely on obscurity.

The crop screen used for uploads runs entirely in the browser and needs no extra setup or libraries.

---

## 8. Test end-to-end

1. **Submit an application.** In `index.html`, fill the form for **Class XI** and choose a stream. Confirm the six subject dropdowns fill with board codes (e.g. "Hindi — 306"), Group-2 won't repeat Group-1, and Electives won't repeat each other. Leave Roll No blank once to confirm that's allowed.
2. **Try the uploads.** For photo, signature, Aadhaar front, Aadhaar back and the Matric certificate (shown for Class XI; TC appears for Class IX), choose a file or use the camera. The crop screen should open and auto-detect the document edges; drag a corner, then press **Crop & Straighten**. Check the preview and size shown under each box. Enter your own email address so you can check step 4.
3. **Check the Sheet.** A new row should appear in `data`, with the six `*FileId` columns filled and the subject columns holding `"SubjectName (Code)"` values.
4. **Check email.** The owner notification (to `OWNER_NOTIFY_EMAIL` or your account) and the applicant confirmation should both show serial number, Application ID, name, class and roll no.
5. **Dashboard login.** Open `admin.html`, log in as `admin` with the `ADMIN_PASSWORD` from step 4. The header should show "Version: … · Logged in as: admin". Click **"देखें" (View)** and confirm the photo, signature and documents load (they're fetched on demand). Click a document to open it full size.
6. **Admin edit — plain fields.** Click **"संपादित करें" (Edit)**, change a field such as the mobile number, and save. The dashboard should update immediately.
7. **Admin edit — files (`admin` account only).** In the same modal you should see upload boxes for photo, signature, Aadhaar front and back, and the TC or Matric box matching the record's class. Pick a new file for one of them: the crop screen opens, then the preview updates. Save and re-open **View** to confirm the new image shows. Change the class field to `Class IX` and confirm the TC box appears and the Matric box disappears.
8. **Non-`admin` check.** If you created a second account ([step 9](#9-optional-set-up-per-user-admin-accounts)), log in as it: the Edit modal should show only the text fields (no upload boxes), and the Export button should be absent.
9. **Download flow.** In `index.html`, use "आवेदन डाउनलोड करें" with the class, DOB and Application ID/PEN. Confirm the masked preview shows **no** images, that it asks for the 12-digit Aadhaar (and rejects a wrong one), and that the correct Aadhaar produces a printable copy **with** photo and signature.
10. **Public edit flow.** Use "आवेदन संशोधित करें" **without** touching any upload, and save. Confirm it updates the existing row (no new row), the stream and subjects reappear in the dropdowns, and all images still display — this exercises the "reuse the existing file" path.

---

## 9. (Optional) Set up per-user admin accounts

By default all staff share one password (`ADMIN_PASSWORD`). To give each person their own login:

1. Open the Sheet's `admins` tab (created automatically on first login, or add it manually with headers `username`, `passwordHash`, `displayName`).
2. Each password must be stored as a **SHA-256 hash, base64-encoded**, matching `hashPassword_()` in `Code.gs`. Easiest: open **`hash-generator.html`** (runs fully offline), enter username, password and display name, click **Generate Hash**, then **Copy Row** — a tab-separated row ready to paste.

   *Alternative:* temporarily add and run this in the Apps Script editor, copy the logged value, then delete the function:
   ```js
   function generateHash() {
     Logger.log(hashPassword_('the-password-you-want'));
   }
   ```
3. Paste the row into `admins`: `username` (e.g. `principal`), `passwordHash` (never the plain password), `displayName` (e.g. `Principal Sir`).
4. Repeat for each staff member.
5. **Once `admins` has any rows, the legacy `ADMIN_PASSWORD` stops working** — every login needs a matching row.
6. **Permissions.** Every account can search, view, edit text fields and delete. Only the account whose `username` is literally **`admin`** (case-insensitive) can **export CSV** or **replace photo, signature, Aadhaar front/back, TC or Matric certificate**. Others never see those controls and are refused server-side if they try. To give a person these abilities, make their row's username `admin`; `displayName` has no effect.

---

## 10. (Optional) Enable SMS notifications

SMS is disabled (`SMS_ENABLED = false`) because it needs a paid third-party gateway.

1. Sign up with an SMS provider (MSG91, Fast2SMS, Twilio, …).
2. In `Code.gs`, fill in the commented `UrlFetchApp.fetch()` call in `sendSmsNotification_()`.
3. Store the API key as a Script Property (e.g. `SMS_API_KEY`), not in the code.
4. Set `SMS_ENABLED = true`.
5. Redeploy (see below).

---

## 11. (Optional) Re-lock files from before private storage

Skip this for a fresh setup.

If an existing deployment used to share photos/signatures publicly, only **new** uploads are private. Files already in the folder keep their public sharing. To lock them, run once from the Apps Script editor:

1. Add a temporary function:
   ```js
   function relockExistingImages_() {
     var folder = getFolder_();
     var files = folder.getFiles();
     var relocked = 0, alreadyPrivate = 0;
     while (files.hasNext()) {
       var file = files.next();
       try {
         file.setSharing(DriveApp.Access.PRIVATE, DriveApp.Permission.NONE);
         relocked++;
       } catch (err) {
         alreadyPrivate++;
       }
     }
     Logger.log('Re-locked: ' + relocked + ', already private/skipped: ' + alreadyPrivate);
   }
   ```
2. Select it in the function dropdown and click **Run**.
3. Check **Executions** for the counts.
4. Every file in the folder is now private. Older applications still work, because `resolveFileId_()` extracts the file ID from the old link column; only the raw Drive link stops working in a browser.
5. Delete the temporary function afterwards (optional).

---

## Updating the backend later

Whenever you edit `Code.gs` after the first deployment:

1. Save your changes.
2. **Deploy → Manage deployments**.
3. Click the pencil/edit icon on your active Web app deployment.
4. Under **Version**, choose **New version**.
5. Click **Deploy**.

The `/exec` URL stays the same, so you don't need to touch `CONFIG.WEB_APP_URL`. Creating a brand-new deployment instead would give a different URL — avoid that unless you want two versions side by side.

**Upgrading from a version without document uploads or `adminUpdate`:** replace `Code.gs`, publish a **New version** (as above), then replace the hosted `index.html` and `admin.html`. Existing sheets pick up the new `aadharFrontFileId`, `aadharBackFileId`, `tcFileId` and `matricFileId` columns automatically on the next write; older rows simply have them blank. **Deploy the new backend before the new `admin.html`**, otherwise Save in the edit modal will fail with "Unknown action".

If you only changed `admin.html` (not `Code.gs`), no Apps Script step is needed — just replace the hosted file. Bump `ADMIN_UI_VERSION` near the top of its script each time; the header displays it next to the username, which tells you whether a browser is running the latest copy or a cached one.

---

## Migrating data from an older spreadsheet-based version

If you have applications in an older, differently-structured spreadsheet:

1. In `Code.gs`, find `importLegacyData_()` near the bottom.
2. Fill in `LEGACY_SHEET_ID` and `LEGACY_SHEET_NAME`.
3. Fill in `COLUMN_MAP`, mapping each new field name (from `HEADERS`) to your old sheet's exact column header. Omit fields your old sheet lacked; they import blank.
4. **If the old sheet stored photo/signature as Drive links or file IDs**, map them under the special `photoUrl` / `signatureUrl` keys — not `photoFileId`/`signatureFileId`. Each triggers `migrateLegacyImage_()`, which makes a **fresh private copy** in this deployment's folder. The original is untouched (if it was public, it stays public). Supporting documents (Aadhaar, TC, Matric) are not migrated by this tool.
5. Select `importLegacyData_` in the function dropdown and **Run**.
6. Check **Executions** for the summary: rows imported, skipped as duplicates, skipped for no Application ID, and any image copies that failed (that row still imports with the field blank).
7. Check the `data` tab, and if you migrated images, open a record in the dashboard and confirm they show via "View".
8. This function is not wired to `doGet`/`doPost`, so it can't be triggered remotely. Re-running is safe: rows whose `applicationId` already exists are skipped.
9. Afterwards you can delete `importLegacyData_` and `migrateLegacyImage_` (optional).

---

## Cleaning up Drive files

The system never deletes Drive files. Deleting a record in the dashboard removes only its sheet row, and replacing a photo or document leaves the previous file in the folder. If storage or tidiness matters, periodically review the Drive folder and remove files for deleted applications (file names start with the Application ID).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `"Server Configuration Error: SHEET_ID is not set"` | Script Property missing | Add `SHEET_ID` in Project Settings → Script Properties |
| `"Unknown action"` on login/submit/**saving an admin edit** | Live deployment runs older code than the editor (e.g. before `adminUpdate` existed) | Deploy → Manage deployments → edit → **New version** → Deploy |
| Login always fails with the right password | `ADMIN_PASSWORD` not set, or the `admins` tab has rows but none match | Check Script Properties or `admins` entries |
| Login works but every action says "Unauthorized"/"session expired" | Token expired (`TOKEN_TTL_MINUTES`, default 60) | Log in again |
| "बहुत अधिक प्रयास" (too many attempts) | Rate limiter / lockout tripped from repeated testing | Wait out `LOGIN_LOCKOUT_MINUTES` / `SEARCH_WINDOW_MINUTES` (counters are keyed by username / search value, not by browser) |
| Upload rejected | Unsupported format | Only JPEG, PNG and WebP are accepted |
| Crop screen doesn't open on upload | Format not allowed (rejected first), or the crop module failed | The original file is used if the module fails; check the browser console. Confirm the file is JPEG/PNG/WebP |
| Auto-detect can't find the document edges | Low contrast, busy background, or document fills the frame | Drag the four corners manually, or use "Full image" / "Standard crop" |
| Document upload ends up very blurry | A large document was reduced to fit ~200 KB | Re-shoot closer/flatter, cropped tightly so fewer pixels are wasted on background |
| Photo/signature/document shows "चित्र लोड नहीं हो सका" in the dashboard | `getImage` failed — admin session expired, or the Drive file was deleted/moved out of `DRIVE_FOLDER_ID` | Log in again; if that doesn't fix it, check the file still exists in the folder |
| Editing an application re-uploads files as new copies every time | Client didn't receive/keep proper file IDs from the edit-load response — usually stale deployment | Redeploy a new version, then retest |
| Pasting an old Drive link in a browser shows "You need access" | Expected once files are private (or re-locked in step 11) | View through the dashboard's "View" button or the applicant's own edit/download flow |
| No owner notification email | `OWNER_NOTIFY_ENABLED` off, wrong `OWNER_NOTIFY_EMAIL`, or the Gmail daily quota is used up | Check the Script Property, the Apps Script **Executions** log, and the owner's Gmail quota |
| Applicant never gets a confirmation email | They left the optional email blank | By design; the owner email always fires |
| No email after an *admin* edit | Admin edits don't send email | Check the `auditLog` tab for the `admin_update` entry |
| CSV export button missing | Logged in as an account other than `admin` | Log in as `admin` |
| Edit modal has no upload boxes | Same restriction — only `admin` can replace files | Log in as `admin` |
| TC or Matric box missing in the edit modal | Shown only when the class field is `Class IX` (TC) / `Class XI` (Matric) | Check the class value in the modal exactly matches |
| CSV export does nothing / errors | Session expired | Refresh and log in again |
| `"Row changed, please refresh and try again"` on delete | Sheet changed between page load and the click | Click 🔄 Refresh, retry |
| Save in the edit modal says "आवेदन नहीं मिला" | Record was deleted or its Application ID changed in the Sheet | Refresh the list |
