# Application Tracker Relay

One-click bridge for logging job applications to the `Application_Tracker` Google Sheet. Bypasses Claude's inability to call arbitrary URLs directly by having Claude produce a JSON blob that is pasted into an HTML tool, which POSTs it through the browser to a Google Apps Script webhook.

## System overview

Two-part system:

1. **Client:** `application-tracker-relay.html` — a stable, single-file HTML tool with a paste-JSON-to-apply workflow and a one-time browser-local configuration panel. Contains no secrets; safe to commit to a public repo.
2. **Server:** Google Apps Script webhook bound to the `Application_Tracker` spreadsheet. Validates a shared token, appends a row, returns a deep-link to the new row.

## First-time setup

The HTML file has no webhook URL or token baked in. On first open:

1. Open `application-tracker-relay.html` in a browser tab (download first — do not run inside the claude.ai iframe; see "Known quirks").
2. The Configuration panel at the top is expanded. The Send button is disabled.
3. Paste the webhook URL into the "Webhook URL" field.
4. Paste the shared token into the "Token" field.
5. Click **Save**. Values are stored in `localStorage` under keys `tracker.webhookUrl` and `tracker.token`.
6. Panel collapses to a masked summary (`script.google.com · token APPT…IOhf`). Send button enables.

Configuration persists until you clear browser data or click **clear** in the config panel. Each browser profile / origin needs its own one-time setup.

## Per-application workflow

1. Provide relevent information to to tool.
2. Tool produces a JSON blob matching the row schema (see "Row JSON schema" below).
3. Open `application-tracker-relay.html` in a browser tab.
4. Paste the JSON into the "Row JSON" textarea and click **Apply to Form** (or presses Ctrl/Cmd+Enter).
5. The form populates. Review and edit inline if needed.
6. Click **Send to Tracker**.
7. On success, click the row-number link to verify the row landed in the sheet.

The HTML file is stable and reused across applications — only the pasted JSON changes. Previous workflows resulted in a 
new artifact being generated every time - a tremendous waste of resources. Now, rather than generating the full contents of this file
with a few lines changed for each application, a small json object is passed and pasted. Necessitated by read-only access to sheets;
created instead of accepted data entry or requesting integration.

## Files

- **`application-tracker-relay.html`** — the client tool. No secrets, no per-application edits. Safe to commit or share publicly.
- **Apps Script** — lives in the Apps Script editor bound to the spreadsheet. Not version-controlled in project files; reference copy below.

## Row JSON schema

Claude produces a JSON object matching this shape per application. The relay accepts either a bare row object or a `{"token": ..., "row": {...}}` wrapper — the bare form is simpler and preferred.

```json
{
  "Company": "...",
  "Role Title": "...",
  "Date Applied": "YYYY-MM-DD",
  "Posting URL": "...",
  "Status": "Preparing | Applied | Interviewing | Offer | Rejected | Withdrawn | No Response",
  "Follow-Up Date": "YYYY-MM-DD or omitted",
  "Resume Version": "...",
  "Cover Letter": "...",
  "Notes": "..."
}
```

Field constraints (enforced both client-side on send and server-side by the Apps Script):

- `Company` and `Role Title` are required.
- `Status` must be one of the seven enum values.
- Date fields must be `YYYY-MM-DD` or empty string.
- **Follow-Up Date auto-default:** if the form field is empty after applying the JSON, it auto-populates to Date Applied + 14 days. To deliberately leave it empty, clear the field with the inline ✕ button after Apply.
- Unknown keys in the JSON are ignored at apply time but surfaced as a warning message so typos get caught.

## Configuration (webhook URL and token)

Both values live in browser `localStorage`, never in the HTML file.

- `tracker.webhookUrl` — the Apps Script web app deployment URL
- `tracker.token` — shared secret, sent in every POST as `payload.token`; must match `SECRET_TOKEN` in the Apps Script

The configuration panel masks the token after saving, showing only the first and last four characters. To view or change values, click **edit** on the collapsed panel. To wipe values entirely, click **clear**.

## Known quirks

### Unable to open Apps Script

If you are logged in to multiple google accounts across multiple tabs, try logging in on an incognito window or another browser.

### Must run outside the claude.ai iframe

When the artifact renders inside claude.ai, the iframe sandbox blocks the cross-origin POST and the request fails with "Failed to fetch." **Always download and open the HTML file directly in a browser tab.** The artifact's error message includes this hint for the network-error path.

### `text/plain` content type

The fetch uses `Content-Type: text/plain` despite sending JSON. This is deliberate: `application/json` triggers a CORS preflight request, and Apps Script web apps do not respond cleanly to preflights. `text/plain` qualifies as a "simple request" that bypasses preflight entirely. Apps Script reads the raw body from `e.postData.contents` and JSON-parses it server-side.

### Apps Script always returns HTTP 200

`ContentService` does not expose any way to set response status codes. Every response is HTTP 200 regardless of success or failure. The only way to detect errors is via `parsed.success === true` in the response body. The client code relies on this exclusively — `res.ok` is meaningless here.

### No response headers controllable

Apps Script also can't set response headers. This is why the CORS dance is as convoluted as it is — there's no server-side `Access-Control-Allow-Origin` to set.

### localStorage scope is per-origin

Opening the file as `file://` vs. from a local web server vs. from a hosted URL each creates a separate storage scope. Switching between them requires re-entering config.

## Verifying the deployment

Open the webhook URL directly in a browser. The `doGet` handler returns a health check:

```json
{"ok":true,"service":"Application Tracker","time":"2026-04-18T..."}
```

If this returns HTML instead of JSON, the deployment is broken or the script isn't deployed as a web app.

## Apps Script source

Reference copy. The live version lives in the Apps Script editor bound to `Application_Tracker`.

```javascript
// ============================================================
// Application Tracker Webhook
// Google Apps Script — bound to Application_Tracker
//
// Deploy as: Web App
//   Execute as: Owner's Google account
//   Who has access: Anyone (no auth)
// ============================================================

const SECRET_TOKEN = "APPTRACKERSECRET-...";  // must match client localStorage token

const COLUMNS = [
  "Company", "Role Title", "Date Applied", "Posting URL", "Status",
  "Follow-Up Date", "Resume Version", "Cover Letter", "Notes"
];

const VALID_STATUSES = [
  "Preparing", "Applied", "Interviewing", "Offer",
  "Rejected", "Withdrawn", "No Response"
];

const DATE_RE = /^(\d{4}-\d{2}-\d{2})?$/;
const STRICT_VALIDATION = true;

function doPost(e) {
  try {
    if (!e || !e.postData || !e.postData.contents) {
      return jsonResponse({ success: false, error: "Empty request body" });
    }
    const payload = JSON.parse(e.postData.contents);
    if (!payload.token || payload.token !== SECRET_TOKEN) {
      return jsonResponse({ success: false, error: "Unauthorized" });
    }
    const row = payload.row;
    if (!row || typeof row !== "object") {
      return jsonResponse({ success: false, error: "Missing or invalid 'row' field" });
    }
    if (STRICT_VALIDATION) {
      if (!row["Company"] || !row["Role Title"]) {
        return jsonResponse({ success: false, error: "Company and Role Title are required" });
      }
      if (row["Status"] && !VALID_STATUSES.includes(row["Status"])) {
        return jsonResponse({
          success: false,
          error: `Invalid status "${row["Status"]}". Must be one of: ${VALID_STATUSES.join(", ")}`
        });
      }
      for (const dateCol of ["Date Applied", "Follow-Up Date"]) {
        const v = row[dateCol];
        if (v && !DATE_RE.test(v)) {
          return jsonResponse({
            success: false,
            error: `Invalid date in "${dateCol}": "${v}". Expected YYYY-MM-DD or empty.`
          });
        }
      }
    }
    const ss = SpreadsheetApp.getActive();
    const sheet = ss.getSheets()[0];
    const values = COLUMNS.map(col => row[col] ?? "");
    sheet.appendRow(values);
    const lastRow = sheet.getLastRow();
    const bg = (lastRow % 2 === 0) ? "#F2F5FB" : "#FFFFFF";
    sheet.getRange(lastRow, 1, 1, COLUMNS.length).setBackground(bg);
    const sheetUrl = ss.getUrl() + "#gid=" + sheet.getSheetId() + "&range=A" + lastRow;
    console.log(`Appended row ${lastRow}: ${row["Company"]} — ${row["Role Title"]}`);
    return jsonResponse({ success: true, rowAppended: lastRow, sheetUrl: sheetUrl });
  } catch (err) {
    console.error(err);
    return jsonResponse({ success: false, error: err.message });
  }
}

function doGet() {
  return jsonResponse({
    ok: true,
    service: "Application Tracker",
    time: new Date().toISOString()
  });
}

function jsonResponse(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}

function testAppend() {
  const sheet = SpreadsheetApp.getActive().getSheets()[0];
  const sample = {
    "Company": "TEST COMPANY",
    "Role Title": "Test Role",
    "Date Applied": new Date().toISOString().split("T")[0],
    "Posting URL": "https://example.com",
    "Status": "Preparing",
    "Follow-Up Date": "",
    "Resume Version": "resume_test",
    "Cover Letter": "coverletter_test",
    "Notes": "DELETE THIS ROW — test only"
  };
  const values = COLUMNS.map(col => sample[col] ?? "");
  sheet.appendRow(values);
  console.log("Test row appended at row " + sheet.getLastRow());
}
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Send button disabled, tooltip says "Save webhook URL and token first" | No config saved in this browser | Fill in the Configuration panel and click Save |
| "Not configured" inline error on send | Same as above, reached via keyboard submit | Same |
| "Failed to fetch" network error | Running inside claude.ai iframe | Download the HTML, open in a fresh browser tab |
| "Unauthorized" in the red box | Token in localStorage doesn't match `SECRET_TOKEN` in Apps Script | Update one of them so they match |
| "Ignored unknown keys" warning after Apply | JSON has keys that don't match the schema (typo or extra fields) | Fix the key names in the JSON |
| "Invalid status" or "Invalid date" error | Malformed value in the form at send time | Fix the value; match the allowed enums/formats |
| `doGet` returns HTML instead of JSON | Script isn't deployed as a web app, or deployment is broken | Redeploy: Deploy → Manage deployments → New version |
| Row appears in sheet but artifact shows error | Response parsing failed | Check browser devtools console; likely an Apps Script runtime error on the response path, not the append path |
| Row landed with wrong column values | `COLUMNS` array in Apps Script doesn't match sheet header row | Align them |
| Config disappeared between sessions | Browser cleared site data, or opened from a different origin (e.g., `file://` vs. localhost) | Re-enter via Configuration panel |

## Rotating the token

1. Pick a new secret string.
2. Update `SECRET_TOKEN` in the Apps Script. Save.
3. Redeploy: Deploy → Manage deployments → pencil → New version → Deploy. URL stays the same.
4. In the client: open the Configuration panel (click **edit**), paste the new token, click Save.
5. Test by sending a row.

No HTML edit needed — the token lives in browser storage, not in the file.
