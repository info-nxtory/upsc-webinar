[README.md](https://github.com/user-attachments/files/30313359/README.md)
# Beyond the Exam — Nxtory UPSC Webinar

Registration, payment, and confirmation system for the "Beyond the Exam" webinar
(₹99, live 26 July 2026, 4:00 PM IST). Live site: `info-nxtory.github.io/upsc-webinar`
(GitHub repo: `info-nxtory/upsc-webinar` — that repo is the source of truth for what's
actually live; the files below are local working copies).

## The flow, end to end

1. Visitor lands on the landing page and clicks "Reserve your seat" → a modal opens.
2. They fill in name / email / mobile and hit "Proceed to Pay."
3. The page POSTs their details to the Apps Script web app. Apps Script:
   - logs a `Pending` row in the Google Sheet
   - asks Razorpay's API to create a **one-time personal payment link** for this
     registrant, tagged with the Sheet row number (via Razorpay's `notes` field)
   - returns that personal link's URL
4. The browser redirects to that personal link (falls back to a shared static
   Razorpay link if link creation fails for any reason, so registration never
   gets stuck).
5. Visitor pays. Two things happen independently:
   - Razorpay redirects their browser to `confirmed.html` (cosmetic — just shows
     a "you're in" page)
   - Razorpay also calls the Apps Script webhook server-to-server with the
     payment details (this is what actually matters)
6. The webhook handler checks a shared-secret token, reads the row number back
   out of the payment's `notes`, marks that exact Sheet row `Confirmed`, records
   the Razorpay payment ID, and sends a confirmation email.
7. On the day of the event, a time-based trigger fires `sendWebinarLinks()`,
   which emails the Google Meet link to everyone marked `Confirmed`.

Registration and payment are matched by the row number tagged on the personal
link (exact), with old-style phone/email matching kept only as a fallback for
the rare case where personal-link creation failed at step 3.

## Files

### Frontend (static, hosted on GitHub Pages)

| File | Purpose |
|---|---|
| `index.html` | Landing page: hero, host section, modal registration form, seat-count check, redirect-to-payment logic. **Currently lives at `/Users/apple/Downloads/index.html`**, not in this folder — it was redesigned and renamed from `upsc-webinar-landing.html`. |
| `confirmed.html` | Static "payment confirmed" page shown after Razorpay redirects back. Purely cosmetic — doesn't verify anything itself. |
| `niveditha-photo.JPG` | Host photo, referenced by `index.html`. Also lives in `/Users/apple/Downloads/`, not this folder. Note the capital `.JPG` — GitHub Pages is case-sensitive, so the filename in the HTML must match exactly. |

### Backend — Google Apps Script (all one Apps Script project, deployed as a Web App)

| File | Purpose |
|---|---|
| `Code.gs` | Shared config only: `SHEET_ID`, `SHEET_NAME`. Deliberately has no functions — see the note below on why. |
| `RazorpayWebhook.gs` | The core of the system. Owns `doPost()`, which routes between two cases: a form submission (writes the Sheet row + creates the personal payment link) and a Razorpay webhook call (confirms payment, matches the row, sends the confirmation email). Also owns `setupRazorpayCredentials()` (one-time setup, stores your Razorpay Key ID/Secret in Script Properties rather than in the file) and the `WEBHOOK_TOKEN` used to authenticate incoming webhook calls. |
| `ConfirmationEmail.gs` | `sendConfirmationEmail()` — sent the moment a payment is confirmed. |
| `EmailTrigger.gs` | `sendWebinarLinks()` — emails the Google Meet link to all `Confirmed` registrants; `setupTrigger()` schedules it to run once, ~3 PM IST on 26 July. |
| `SeatCounter.gs` | Owns `doGet()`. Returns how many seats are `Confirmed` so the landing page can show "X seats left" or close registration at 98. |
| `FixRedirect.gs` / `FixRedirect_1.gs` | One-off scripts used once to fix a Razorpay payment link's redirect URL. Not part of the live request flow — safe to delete, kept only as a record of what was run. |

### Google Sheet columns

`A` Timestamp · `B` Name · `C` Email · `D` Phone · `E` Payment Status (`Pending`/`Confirmed`) · `F` Razorpay Payment ID · `G` Meet Link Sent (`Yes`/`No`)

## Things worth knowing if you're picking this up later

- **Only one `doPost` and one `doGet` may exist across all files.** Apps Script
  silently keeps just one definition when a function name is declared twice,
  with no error — this broke payment confirmation once already. `doPost` lives
  only in `RazorpayWebhook.gs`; `doGet` lives only in `SeatCounter.gs`.
- **Apps Script's `doPost(e)` cannot read custom HTTP headers** (Google has
  confirmed this isn't supported), so Razorpay's usual signature-header
  verification doesn't work here. The webhook is instead authenticated with a
  `?token=...` query parameter on the webhook URL configured in Razorpay's
  dashboard, checked against `WEBHOOK_TOKEN` in `RazorpayWebhook.gs`.
- **Razorpay credentials are stored in Script Properties**, not hardcoded in
  any file — run `setupRazorpayCredentials()` once from the Apps Script editor
  after pasting in your Key ID/Secret, then blank them back out.
- **The client fetch to Apps Script uses `Content-Type: text/plain`**, not
  `application/json` — this avoids a CORS preflight request that Apps Script
  doesn't handle, which otherwise silently breaks the response.
- **A `LockService` lock guards the Sheet row-append** in `handleFormSubmission`,
  so concurrent registrations can't read back the wrong row number and tag the
  wrong person's payment link.
- **GitHub Pages filenames are case-sensitive** — a mismatch between
  `niveditha-photo.jpg` and `niveditha-photo.JPG` broke the host photo once.

## Deploying a change

**Apps Script files** (`.gs`):
1. Edit the file locally, then select-all/paste the new content into the
   matching file at script.google.com.
2. **Deploy → Manage deployments → Edit → New version → Deploy.** Saving alone
   does not update the live `/exec` URL — this step is easy to miss.

**HTML/image files**:
1. Edit locally, then upload via the GitHub repo's **Add file → Upload files**,
   overwriting the existing file of the same name.

## Known limitations

- If personal payment link creation fails (e.g. Razorpay API hiccup), the
  system falls back to a shared static link with phone/email matching, which
  can occasionally create a duplicate-looking Sheet row instead of updating
  the original `Pending` one.
- `confirmed.html` doesn't cryptographically verify anything — it's shown to
  anyone who lands on that URL, including by direct navigation. The Sheet and
  confirmation email (via the webhook) are the actual source of truth for
  whether someone paid.
