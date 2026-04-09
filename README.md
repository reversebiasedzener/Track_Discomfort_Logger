# Track Ride Monitor — Setup Guide

A mobile Progressive Web App for measuring railway track ride quality.
Records X/Y/Z accelerations + GPS at 5 readings/second, only above 80 km/h.
Syncs to Google Sheets. Discomfort button flags a ±5 second window.

---

## Files in this package

| File | Purpose |
|------|---------|
| `index.html` | The complete mobile app (PWA) |
| `manifest.json` | PWA install manifest |
| `sheet-backend.gs` | Google Apps Script — paste into your Sheet |

---

## Step 1 — Deploy the Google Apps Script backend (~10 minutes)

1. Create a new Google Sheet (name it e.g. "Track Ride Data")
2. Click **Extensions → Apps Script**
3. Delete any existing code in the editor
4. Open `sheet-backend.gs` from this package and paste the entire contents
5. Click **Deploy → New Deployment**
6. Set:
   - **Type**: Web App
   - **Execute as**: Me
   - **Who has access**: Anyone
7. Click **Deploy** → grant permissions when prompted
8. Copy the **Web App URL** (looks like `https://script.google.com/macros/s/ABC123.../exec`)
9. Keep this URL — you'll enter it into the app

### First-run: add conditional formatting (optional but useful)
In Apps Script editor: **Run → addConditionalFormatting**
This colour-codes high acceleration values automatically.

---

## Step 2 — Deploy the app (~5 minutes)

### Option A: GitHub Pages (free, recommended)
1. Create a GitHub account if needed
2. Create a new repository (e.g. `track-monitor`)
3. Upload `index.html` and `manifest.json`
4. Go to **Settings → Pages → Source: main branch**
5. Your app URL: `https://yourusername.github.io/track-monitor`

### Option B: Any static web host
Upload `index.html` and `manifest.json` to any web server.
Must be served over **HTTPS** for GPS and motion sensors to work.

### Option C: Local testing (Android only)
- Use [http-server](https://www.npmjs.com/package/http-server) on your PC
- Connect phone to same WiFi
- Access via IP — Android allows DeviceMotion on local network

---

## Step 3 — Install on phone

### Android (Chrome)
1. Open the app URL in Chrome
2. Tap **⋮ → Add to Home Screen**
3. It installs like a native app

### iPhone (Safari)
1. Open the app URL in **Safari** (not Chrome — iOS only allows sensors in Safari)
2. Tap **Share → Add to Home Screen**
3. When you open the app, tap **⚙ Setup** first — this triggers the motion permission dialog

---

## Step 4 — Configure the app

1. Open the app → tap **⚙ Setup**
2. Paste the Web App URL from Step 1
3. Enter your session/run name (e.g. "Delhi–Agra 15-Jan Run 1")
4. Speed threshold: leave at 80 (or adjust as needed)
5. Tap **Save & Close**

---

## Using the app

| Action | What happens |
|--------|-------------|
| **Start Session** | Begins recording. Shows waiting screen until speed ≥ 80 km/h |
| **Train reaches 80 km/h** | Recording starts automatically, data flows to Sheets |
| **MARK DISCOMFORT** (red button) | Flags the current moment. Retroactively flags 2s of prior data + 3s ahead. Inserts a PEAK_SUMMARY row in Discomfort sheet |
| **Train drops below 80** | Recording pauses, waiting screen returns |
| **Stop Session** | Ends recording |

---

## Google Sheets structure

### RawLog sheet
Every 200ms reading above 80 km/h:

| Column | Description |
|--------|-------------|
| Timestamp | ISO 8601 UTC |
| Date | yyyy-MM-dd |
| Time | HH:mm:ss.SSS |
| Session | Run name you entered |
| Latitude / Longitude | GPS coordinates |
| Speed_kmh | GPS-derived speed |
| Ax_ms2 (Lateral) | Side-to-side sway |
| Ay_ms2 (Vertical) | Bounce/heave |
| Az_ms2 (Longitudinal) | Braking/acceleration jerk |
| Resultant_ms2 (XZ) | √(Ax²+Az²) — combined ride harshness |
| Discomfort_Flag | YES if within flagged window |

### Discomfort sheet
One row per discomfort event, showing peak values in the ±5s window.

### Sessions sheet
Summary: session name, first/last timestamp, discomfort event count.

---

## Benchmark analysis in Sheets

Once you have comfortable-run data and discomfort-flagged data:

### Find your comfort baseline
In a new sheet, add this formula to compute 90th percentile of resultant from a known-good session:
```
=PERCENTILE(FILTER(RawLog!K2:K, RawLog!D2:D="YourGoodSessionName"), 0.9)
```

### Flag anything above that threshold
Use conditional formatting on column K (Resultant) with:
- Rule: **Greater than** → `=PERCENTILE(FILTER(...)…, 0.9)`
- Format: red background

### Compare sessions visually
Insert → Chart → X axis: Time column, Y values: Ax, Az, Resultant
Use the "Discomfort_Flag = YES" column as a filter to overlay events.

---

## Sensor orientation

Hold the phone in **portrait mode** with the screen facing you:
- **X (Lateral)**: side-to-side sway of the coach
- **Y (Vertical)**: up-down bounce (resting ~9.81 m/s² = gravity)
- **Z (Longitudinal)**: fore-aft jerk (braking/acceleration)

For consistent readings, mount the phone in a holder rather than holding it by hand.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Sync failed" in app | Check Sheets URL, ensure "Anyone" access in deployment |
| Speed stays at 0 | Allow location permission; GPS needs clear sky view |
| No accelerometer data (iOS) | Tap ⚙ Setup first — this triggers the permission prompt |
| Data not appearing in Sheets | Check Apps Script execution log (View → Logs) |
| App not installing as PWA | Must be served over HTTPS; use GitHub Pages |

---

## Privacy note
All data stays within your Google account. The app communicates only with:
1. The Google Sheets URL you configure
2. Google Fonts (for typography)
No other servers receive your location or sensor data.
