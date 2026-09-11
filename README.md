# 🌴 ThaiSplit - Group Expense Tracker

ThaiSplit is a lightweight, mobile-first, serverless web application designed to seamlessly track, calculate, and settle group expenses during trips. It runs entirely in the browser using Vanilla JavaScript and uses Google Sheets as a free, scalable cloud database.

## ✨ Key Features

* **📱 Mobile-First UI**: A clean, fluid, glassmorphism design built with Tailwind CSS, optimized exclusively for mobile touch interfaces.
* **☁️ Smart Cloud Sync**: Uses a custom "Merge Engine" to prevent data overwrites when multiple friends add expenses at the exact same time.
* **📴 Offline-First Capability**: Log expenses on a ferry with zero cell service. The app caches data locally and automatically syncs to the cloud the second you regain connection.
* **💱 Dual Currency Support**: Log transactions in either Thai Baht (฿) or Indian Rupees (₹). The app automatically handles live conversion (`1 THB = 2.92 INR`) and unifies all balances.
* **🧮 Flexible Splitting**: Choose between splitting a bill **Equally** among selected travelers, or entering **Exact** custom amounts.
* **⚖️ Debt Simplification Engine**: Uses a greedy algorithm to calculate net balances and output the absolute minimum number of transactions needed for everyone to settle up.
* **🕒 Activity Audit Trail**: A dedicated feed that tracks exactly who added, edited, deleted, or settled transactions (with timestamps).
* **🔒 Protected Actions**: Built-in PIN protection for destructive actions (deleting expenses or resetting the trip).

---

## 🛠️ Tech Stack

* **Frontend**: Pure HTML5, CSS3, Vanilla JavaScript (ES6+).
* **Styling**: Tailwind CSS (via CDN).
* **Database / Backend**: Google Sheets API via Google Apps Script (Serverless).
* **Storage Engine**: Browser `localStorage` + Chunked Google Sheets JSON storage.

---

## 🚀 Setup & Deployment

Because ThaiSplit uses Google Sheets as a database, you need to configure the Google Apps Script backend before hosting the frontend.

### Step 1: Set up the Google Sheets Database
1. Go to [Google Sheets](https://sheets.google.com) and create a new Blank Spreadsheet. Name it "ThaiSplit Database".
2. In the top menu, click **Extensions > Apps Script**.
3. Delete any code in the editor and paste the backend script below. This script safely slices the JSON data into 40,000-character chunks to bypass Google's 50k-character cell limit.

```javascript
// Handles fetching the data (GET)
function doGet(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
  var lastRow = sheet.getLastRow();
  var fullJsonString = "";
  
  if (lastRow > 0) {
    var values = sheet.getRange(1, 1, lastRow, 1).getValues();
    for (var i = 0; i < values.length; i++) {
      if (values[i][0]) fullJsonString += values[i][0];
    }
  }
  
  return ContentService.createTextOutput(fullJsonString || JSON.stringify({}))
    .setMimeType(ContentService.MimeType.JSON);
}

// Handles saving the data (POST) with LockService for Race Conditions
function doPost(e) {
  var lock = LockService.getScriptLock();
  if (lock.tryLock(10000)) {
    try {
      var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
      var payload = e.postData.contents;
      var chunkSize = 40000;
      var chunks = [];
      
      for (var i = 0; i < payload.length; i += chunkSize) {
        chunks.push([payload.substring(i, i + chunkSize)]);
      }
      
      sheet.clear(); 
      sheet.getRange(1, 1, chunks.length, 1).setValues(chunks);
      
      return ContentService.createTextOutput(JSON.stringify({ status: "success" }))
        .setMimeType(ContentService.MimeType.JSON);
    } finally {
      lock.releaseLock();
    }
  } else {
    return ContentService.createTextOutput(JSON.stringify({ error: "Server busy" }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

4. Click **Deploy > New deployment**.
5. Choose **Web app**. Set *Execute as* to `Me` and *Who has access* to `Anyone`.
6. Authorize the script and copy the generated **Web app URL**.

### Step 2: Connect the Frontend
1. Open the `index.html` file in this repository.
2. Locate the `GOOGLE_SCRIPT_URL` variable.
3. Replace the placeholder string with the Web app URL you copied in Step 1:
   ```javascript
   const GOOGLE_SCRIPT_URL = '[https://script.google.com/macros/s/YOUR_UNIQUE_ID_HERE/exec](https://script.google.com/macros/s/YOUR_UNIQUE_ID_HERE/exec)';
   ```

### Step 3: Host on GitHub Pages
1. Push your updated `index.html` file to your GitHub repository.
2. Go to your repository **Settings > Pages**.
3. Under *Build and deployment*, set the branch to `main` (or `master`) and save.
4. Your app is now live! 

---

## 🔐 Admin Controls & Security

To prevent accidental data loss during the trip, the app uses soft-PIN protection:
* **Delete an Expense:** Requires PIN (This performs a "soft delete" to maintain sync integrity).
* **Reset Entire Trip:** Requires Admin Password

---

## ⚠️ Important Notes for Developers
* **Cache-Busting:** The `<head>` of the HTML document contains strict no-cache meta tags. This ensures that if you push an update to GitHub Pages, users will immediately see the latest version upon refreshing, without old JavaScript interfering.
* **Data Schema Adjustments:** If you change the data schema significantly, update the `STORAGE_KEY` variable (e.g., `thaiTripData_v8`) in the JavaScript. This forces client browsers to ignore their old local data structure and perform a clean fetch from the cloud.
