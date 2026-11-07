# UMV Chochahi Chhapra — Online Admission System

A self-contained online admission system for **UMV Chochahi Chhapra, Paroo, Muzaffarpur** (UDISE: 10140301702), built entirely on free Google infrastructure — Google Sheets as the database, Google Drive for file storage, and Google Apps Script as the backend. No paid hosting, no external database, no API keys required for the core system.

The project has three main parts plus an optional utility:

| File | Role |
|---|---|
| `Code.gs` | Backend — Google Apps Script Web App (API, data storage, auth, email) |
| `index.html` | Public admission form — apply, edit, download/print an application |
| `admin.html` | Staff dashboard — search, view, edit, delete, export applications; replace photo/signature/documents (`admin` account only) |
| `hash-generator.html` | Optional offline utility — creates password-hash rows for per-user staff logins |

For deployment steps, see **[SETUP_GUIDE.md](./SETUP_GUIDE.md)**.
For how the pieces fit together, see **[ARCHITECTURE.md](./ARCHITECTURE.md)**.

---

## Features

### For applicants (`index.html`)

- **Apply** — a full admission form: class / roll no. / stream / subjects, identity numbers (PEN, APAAR, Aadhaar, e-Shiksha, OTR), personal and family details, address, health details, previous school, bank details (for scholarship transfer), and uploads. **Roll No** is optional.
- **Class-dependent rules** — Class I applicants are exempt from PEN, APAAR, Aadhaar, previous-school UDISE and the Aadhaar-card uploads. Class IX/X shows a language-group selector. Class XI/XII shows stream and subject selection.
- **Class XI/XII subject selection matches the official BSEB code sheet exactly** — picking a stream (Arts/Science/Commerce) fills dropdowns for Compulsory Group-1, Compulsory Group-2, three Electives, and an optional Additional subject, each showing the board's own numerical code (e.g. "Hindi — 306"). The dropdowns enforce the board's rules automatically (Group-2 can't repeat Group-1, Electives can't repeat each other, Additional can't repeat anything already chosen), so an invalid combination can't be submitted.
- **Uploads, with auto crop & straighten** — photo, signature, and supporting documents:
  - Student photo and signature (always required for a new application).
  - **Aadhaar card front and back** (required, except for Class I where they're optional).
  - **Transfer Certificate (TC)** — shown and required only for **Class IX**.
  - **Matric certificate** — shown and required only for **Class XI**.
  - Each upload can be chosen from the device or taken with the camera. Before anything is saved, a crop screen opens that **auto-detects the document's four corners and flattens the perspective**; the applicant can drag the corners, rotate, use a standard rectangular crop, or skip to the full image.
  - Photo and signature are then resized/compressed to fixed targets (photo 50–100 KB, signature 5–20 KB). Documents keep their aspect ratio and are compressed to about 200 KB.
- **Preview before submitting** — a modal shows exactly what will be sent.
- **Edit a submitted application** — re-verify with Aadhaar + mobile number, then load and update the original record (an edit updates the existing row; no duplicate is created). Existing photo, signature and documents are reused unless the applicant replaces them.
- **Download / print an application** — generates a printable, styled copy in a new window (usable as "Print to PDF"). The window has a **Download PDF** button that creates the one-page A4 PDF directly (works on Android and iPhone, where the print dialog has no paper-size or pages-per-sheet options) and a **Print** button. It is laid out to fit **one A4 page**: fields are in a compact two-pair-per-row table, the page size is fixed to A4 with no browser header/footer, and the content is scaled down automatically if it would otherwise overflow. The PDF button loads two small libraries (html2canvas, jsPDF) from cdnjs, so it needs an internet connection; the PDF is an image of the page, so its text can't be selected.
- **Self-lookup with masked preview** — search by class + DOB + (Application ID or PEN) without logging in; the result is a **masked preview** (most digits of Aadhaar/mobile/bank details hidden, no images) that the applicant must confirm before the **full** record is released. Confirming also requires typing the applicant's own 12-digit Aadhaar number.
- **One application per Aadhaar** — duplicate submissions on the same identity number are rejected (edit is offered instead).
- **Lightweight arithmetic CAPTCHA** on the edit-lookup and search actions, on top of server-side rate limiting.
- **Confirmation email** — sent only if the applicant filled in the optional email field; contains serial number, Application ID, student name, class, and roll no.

### For staff (`admin.html`)

- **Username/password login** with signed, expiring session tokens (the raw password is never resent after login).
- **Search & filter** by class and free text (name, Application ID, PEN, mobile, Aadhaar), paginated server-side.
- **View full record** in a modal. Photo, signature and uploaded documents are fetched on demand when the modal opens (the files are private and only released to an authenticated admin session). Click a document to open it full size.
- **Edit** any record's plain data fields (fixing a typo in a name, mobile, address, class, etc.) without going through the public Aadhaar+mobile flow. Every logged-in admin account can do this. The Application ID can't be changed.
- **Replace photo, signature, Aadhaar front, Aadhaar back, TC (Class IX) or Matric certificate** from the same edit modal — **restricted to the account named `admin`**. Other accounts don't see the upload boxes and are refused server-side if they try. Uploads use the **same auto crop & straighten screen** and the same size targets as the public form. The TC box appears only when the record's class is Class IX, the Matric box only for Class XI.
- **Delete** a record (confirmation prompt + stale-row safety check).
- **CSV export** of the full filtered result set — also restricted to the `admin` account.
- **Session auto-expiry handling** — any API call that reports an invalid/expired session returns the dashboard to the sign-in screen.
- **Header shows the dashboard version and who's logged in** (e.g. "Version: v3-2026-11-05 · Logged in as: admin") — useful for confirming a browser is running the latest `admin.html`.

### Backend (`Code.gs`)

- **Google Sheet as database** — a `data` tab holds every application; columns are looked up by header name everywhere, so new fields can be appended to `HEADERS` and old sheets migrate automatically.
- **Google Drive for files, kept private** — photos, signatures and documents are decoded from base64, validated (JPEG/PNG/WebP only) and stored as separate files. Files are **not** publicly shared. Bytes are only handed out embedded in an API response at the moment the requester has earned the right to see them — never as a standalone URL. See [ARCHITECTURE.md](./ARCHITECTURE.md).
- **Admin accounts** — an optional `admins` tab (username / SHA-256 password hash / display name), with automatic fallback to a single legacy shared password (`ADMIN_PASSWORD` script property) if that tab is empty.
- **Brute-force protection** — exponential backoff on failed logins, temporary lockout after repeated failures, and sliding-window rate limits on the public search / edit-lookup / reveal endpoints.
- **Append-only audit log** — an `auditLog` tab records logins, failed logins, exports, deletions, admin updates (including which files were replaced) and public search reveals.
- **Owner/staff notification email** — sent on *every* submission and edit, regardless of whether the applicant gave an email address.
- **SMS notifications** — stubbed out and disabled by default.
- **One-off legacy data importer** — a function not wired to any endpoint, for migrating from an older spreadsheet-based version.

---

## Tech stack

- **Backend:** Google Apps Script (`doGet` / `doPost` Web App)
- **Database:** Google Sheets
- **File storage:** Google Drive
- **Email:** Apps Script `MailApp`
- **Frontend:** Plain HTML/CSS/vanilla JavaScript — no build step, no frameworks, no dependencies
- **Fonts:** Google Fonts (Noto Sans Devanagari + Inter) via CDN link

---

## Project structure

```
.
├── Code.gs               # Apps Script backend — deploy as a Web App
├── index.html            # Public admission form (apply / edit / download)
├── admin.html            # Staff admin dashboard
├── hash-generator.html   # Optional offline utility — generates admins-sheet password hashes
├── README.md
├── ARCHITECTURE.md
└── SETUP_GUIDE.md
```

---

## Quick start

1. Follow **[SETUP_GUIDE.md](./SETUP_GUIDE.md)** end to end — it covers creating the Sheet and Drive folder, setting Script Properties, deploying the Web App, and pointing both HTML files at the deployed URL.
2. Open `index.html` to test the public form.
3. Open `admin.html` and log in to test the dashboard.
4. (Optional) For individual staff logins instead of one shared password, use `hash-generator.html` (see [SETUP_GUIDE.md, step 9](./SETUP_GUIDE.md#9-optional-set-up-per-user-admin-accounts)).

---

## Who can do what

| Action | Any admin account | `admin` account only |
|---|---|---|
| Search, view records and images | ✅ | ✅ |
| Edit plain data fields | ✅ | ✅ |
| Delete a record | ✅ | ✅ |
| Replace photo / signature / Aadhaar front & back / TC / Matric | ❌ | ✅ |
| Export CSV | ❌ | ✅ |

The restriction is enforced server-side in `Code.gs`; hiding the buttons in `admin.html` is only a convenience.

---

## Data privacy notes

- Aadhaar, bank account, and mobile fields are **masked** in the unauthenticated public search preview. The full record (including photo and signature) is released only after a second step: a short-lived signed token plus the applicant typing their own 12-digit Aadhaar number.
- **All uploaded files on Drive are private**, not "anyone with the link." There is no standalone URL for them; bytes are only embedded in a response the requester has already earned, or fetched by an admin on demand through a token-checked API call.
- Supporting documents (Aadhaar card images, TC, Matric certificate) are blanked from the public masked preview along with the photo and signature. The public "download application" copy shows photo and signature only; documents are viewable by the applicant in the edit flow and by staff in the dashboard.
- **Applications submitted before private storage was introduced** keep working: the backend recognizes the old public link, extracts the file ID and serves the image privately from then on. However, the *original* public-link files stay publicly linkable unless re-locked — see SETUP_GUIDE.md step 11.
- A CSV export's file columns hold private Drive file IDs, not clickable links; they're only useful through the dashboard's "View" button.
- **Deleting a record removes the sheet row only.** Its files stay in the Drive folder, as do files superseded by a replacement upload. Clean these up manually in Drive if needed.

---

## License

Internal use for UMV Chochahi Chhapra. Adapt freely for your own school's admission process.
