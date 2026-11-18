// ============================================================
// Shakila & Isma Wedding RSVP — Google Apps Script backend
//
// SETUP (one time, about 5 minutes):
// 1. Go to https://script.google.com and click "+ New project".
// 2. Delete the sample code, paste ALL of this file in.
// 3. Click "Save" (floppy icon), name it "Wedding RSVP Backend".
// 4. Click "Deploy" > "New deployment" > gear icon > choose "Web app".
//    - Description: rsvp
//    - Execute as: Me
//    - Who has access: ANYONE        <-- important!
// 5. Click "Deploy", approve the permissions (choose your account,
//    "Advanced" > "Go to ... (unsafe)" > "Allow" — it is your own script).
// 6. Copy the "Web app URL" (ends in /exec).
// 7. Open index.html, paste that URL into BACKEND_URL at the top of
//    the <script> section, save, and redeploy the site.
//
// Replies are stored in a Google Sheet created automatically.
// Open https://docs.google.com/spreadsheets to see the data.
// ============================================================

function doGet(e) {
  const action = (e && e.parameter && e.parameter.action) || "";
  if (action === "list") {
    return json({ guests: readAll() });
  }
  return json({ ok: true });
}

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents);
    if (body.action === "add" && body.reply) {
      const r = body.reply;
      appendRow([
        new Date(),
        String(r.name || "").slice(0, 80),
        r.coming ? "YES" : "NO",
        Number(r.adults) || 0,
        Number(r.kids) || 0
      ]);
      return json({ ok: true });
    }
    return json({ ok: false, error: "unknown action" });
  } catch (err) {
    return json({ ok: false, error: String(err) });
  }
}

// ---- storage helpers (auto-creates the sheet) ----

function getSheet() {
  const name = "RSVP Replies";
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(name);
  if (!sheet) {
    sheet = ss.insertSheet(name);
    sheet.appendRow(["Timestamp", "Name", "Coming", "Adults", "Kids"]);
    sheet.setFrozenRows(1);
  }
  return sheet;
}

function appendRow(values) {
  // Lock prevents two guests submitting at the same moment from clashing
  const lock = LockService.getScriptLock();
  lock.waitLock(10000);
  try {
    getSheet().appendRow(values);
  } finally {
    lock.releaseLock();
  }
}

function readAll() {
  const sheet = getSheet();
  const rows = sheet.getDataRange().getValues();
  const guests = [];
  for (let i = 1; i < rows.length; i++) {
    const r = rows[i];
    guests.push({
      name: r[1],
      coming: r[2] === "YES",
      adults: Number(r[3]) || 0,
      kids: Number(r[4]) || 0
    });
  }
  return guests;
}

function json(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
