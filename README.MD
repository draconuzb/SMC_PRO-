# 🔍 Boshqaruvchi — Full Expert Audit Report
**Audit Date:** 21 March 2026 | **Auditor:** AI Code Review

---

## 📊 Project Overview

| Component | Files | Lines of Code | Status |
|-----------|-------|--------------|--------|
| Telegram Bot Core ([bot.js](file:///tmp/ns_bot.js)) | 1 | 1,650 | ✅ Running |
| Bot Scenes ([scenes.js](file:///tmp/ns_scenes.js)) | 1 | 1,145 | ✅ Running |
| CEO Reports (`ceo_scenes.js` + `ceo_reports.js`) | 2 | 1,480 | ✅ Running |
| TWA API ([twa_api.js](file:///tmp/ns_twa_api.js)) | 1 | 660 | ✅ Running |
| WebApp Frontend (`app.js`) | 3 versions | ~5,000 | ⚠️ 2 dead files |
| AI Module (`ai.js`) | 1 | 190 | ❌ Disabled |
| Notion Module (`notion.js`) | 1 | 80 | ❌ Disabled |

---

## 🚨 SECTION 1: Features That Are NOT Working

### 1.1 AI / Gemini Intelligence — Completely Dead
**Severity: Critical | Impact: High**

The entire `ai.js` module (Gemini 2.0 Flash integration) is fully coded with 3 real capabilities:
- `analyzeReport()` — analyzes reports with AI insights
- `detectAnomalies()` — detects anomalies in school data
- `askQuestion()` — answers natural language questions

**Why it fails:** `GEMINI_API_KEY` is not in `.env`. The module gracefully disables itself but is **never called anywhere in the bot** — not in `ceo_reports.js`, not in [scenes.js](file:///tmp/ns_scenes.js), not in [bot.js](file:///tmp/ns_bot.js). The code exists but is completely orphaned.

**What the user is missing:** Every report could automatically show an AI summary, anomaly alerts, and actionable insights. This is the most powerful feature in the codebase and it is completely inactive.

---

### 1.2 Notion Integration — Completely Dead
**Severity: High | Impact: Medium**

`notion.js` is coded to sync rejection data to a Notion database. It is not called anywhere in the current bot code ([scenes.js](file:///tmp/ns_scenes.js), [bot.js](file:///tmp/ns_bot.js)). The `NOTION_TOKEN` and `NOTION_DATABASE_ID` are not in `.env`.

**What the user is missing:** Rejection entries were supposed to be automatically synced to a Notion database for external tracking.

---

### 1.3 WebApp: No Reports/Hisobotlar Section
**Severity: High | Impact: High**

The WebApp (`app.js`) has these tabs: Home, Lead, Debtors, Finance, Attendance, Problems, Rejections, Rooms, Reports, Users.

The **Reports tab** exists in the nav but its content is:
- Fetches combined data from `/api?endpoint=report`
- Renders a basic text-style table

**Missing in WebApp:**
- No tax report viewer
- No PDF download button
- No weekly/monthly aggregated chart view
- No anomaly alerts section
- No AI insights display (even though it's built in backend)

---

### 1.4 WebApp: Finance Tab Missing Expense Type Breakdown
**Severity: Medium | Impact: Medium**

The Finance tab shows total income/expense but does NOT show:
- A breakdown by expense type (e.g. "Salary: 5,000,000 | Utilities: 500,000")
- A visual chart of Rasmiy vs Norasmiy income/expense split
- Monthly trend chart (Chart.js is imported but barely used)

The API already returns all this data in the `byCategory` and `expensesByType` fields — the frontend just doesn't render it.

---

### 1.5 `/addmanager` Command vs WebApp User Add
**Severity: Medium | Impact: Medium**

There is a `/addmanager` command in the bot (now fixed to save to DB). However, when the user is added via `/addmanager`, they get **all permissions set to 0** (no sections enabled). The Admin must then go to the WebApp user management to assign permissions. There is no guided permission-assignment flow directly from the bot. This creates an awkward two-step onboarding process.

---

### 1.6 Search Scene — Month Mapping Bug
**Severity: Medium | Impact: Medium**

In `search_scene.js`, the month label is matched against a local array of 6 month names. However, if the system uses the "Sentyabr" spelling (in `utils.js`) but the search scene uses "Sentyabr" — these match. But [twa_api.js](file:///tmp/ns_twa_api.js) uses "Sentabr" (no 'y'). This **inconsistency in Uzbek month spelling** across files causes month-matching to silently fail for September data.

**Files affected:** `utils.js`, [twa_api.js](file:///tmp/ns_twa_api.js), [storage.js](file:///tmp/ns_storage.js) — all have slightly different month name arrays.

---

### 1.7 PDF Generator — Sent to `/tmp`, Not Cleaned Up
**Severity: Low | Impact: Low**

`pdf_generator.js` saves PDFs to `/tmp`. The code has `fs.unlinkSync(pdfPath)` after sending, which is correct. But if the send fails mid-way (network error), the cleanup never runs, leaving orphaned PDFs in `/tmp`. On a shared host with limited disk, this can accumulate over time.

---

### 1.8 Accountability Cron — False Positives Possible
**Severity: Low | Impact: Low**

The `accountability` cron at 21:00 checks if managers submitted a report "today." However, it checks against SQLite data using `substr(timestamp,1,10)`. If a manager submitted data yesterday but not today, they get flagged. But if they submitted earlier in the day and the field was deleted, there's no "did they actually report today" tracking — it checks if ANY data exists for today's date across all tables, which can be misleading.

---

## 🔧 SECTION 2: Features Not Yet Implemented (But Should Be)

### 2.1 AI Features — Plugin Ready, Never Activated
**Priority: URGENT**

**What needs to happen:**
1. Add `GEMINI_API_KEY=your_key` to `.env`
2. In `ceo_reports.js`, after generating each report text, call `ai.analyzeReport(text, type, lang)` and append the result
3. Add an "🤖 AI Tahlil" button to the report menu in `bot.js`
4. Wire `ai.detectAnomalies()` into the morning summary cron

This is the single highest-value feature not activated.

---

### 2.2 WebApp: Charts and Data Visualization
**Priority: HIGH**

Chart.js is already imported in `index.html`. The data is already returned by the API. But no charts are rendered anywhere.

**Should be added:**
- Line chart of lead count per day (last 30 days) on the Home tab
- Pie chart of Rasmiy vs Norasmiy income on Finance tab
- Bar chart of debtors by month on Debtors tab
- Attendance percentage gauge on Attendance tab

---

### 2.3 WebApp: Real-Time Notifications / Push
**Priority: MEDIUM**

The WebApp has no live update mechanism. If a manager submits data, the CEO must manually refresh to see it. Options:
- Add a WebSocket or Server-Sent Events endpoint to the Node.js API
- OR add a simple "auto-refresh every 60 seconds" toggle

---

### 2.4 WebApp: Tax Report Viewer
**Priority: MEDIUM**

`tax_reports.js` generates Foyda Soligi and Daromad Soligi reports beautifully in the bot. But the WebApp has no tax tab or section. The TWA API has no `/api?endpoint=tax` endpoint either. Users who prefer the WebApp cannot see tax reports at all.

**What needs to be added:**
- A GET endpoint `?endpoint=tax_report&month=Mart 2026` in `twa_api.js`
- A new "Soliq" tab in `app.js`

---

### 2.5 WebApp: CSV / PDF Export Button
**Priority: MEDIUM**

`export.js` (CSV) and `pdf_generator.js` (PDF) are both fully functional. But the WebApp has no "Export" or "Download" button. Only the Telegram bot can trigger exports. This means the CEO cannot download a spreadsheet from the WebApp dashboard.

**What needs to be added:**
- POST endpoint `?endpoint=export_csv` in `twa_api.js`
- Download button on the Reports tab of the WebApp

---

### 2.6 User Onboarding Flow — Permission Assignment in Bot
**Priority: MEDIUM**

When `/addmanager` adds a new user, a full inline-keyboard permission picker should appear so the admin can immediately set which sections the new user can access, without having to open the WebApp separately.

---

### 2.7 Notion Sync — Activation
**Priority: LOW**

The Notion integration just needs:
1. Add `NOTION_TOKEN` and `NOTION_DATABASE_ID` to `.env`
2. Import and call `notion.addRejection(ts, fan, count)` inside the rejection wizard step in `scenes.js`

---

### 2.8 Data Deletion — Missing "Are you sure?" for Finance Records
**Priority: LOW**

Delete confirmations exist for `davomat`, `lead`, and `qarzdorlar`. But **Finance record deletion** (`moliya_kirim_rasmiy_del` and `moliya_kirim_norasmiy_del`) goes through a scene, but there is **no final confirmation step** before the actual database DELETE runs. A wrong tap permanently deletes financial data.

---

### 2.9 WebApp: Multi-User Activity Indicator
**Priority: LOW**

The WebApp shows no indication of which managers submitted data today. The CEO has to generate a full report just to see who reported. A simple "Today's Activity" widget on the home dashboard showing each manager's submission status would be very useful.

---

## ⚠️ SECTION 3: Architecture & Infrastructure Issues

### 3.1 Three Duplicate WebApp Versions
**Severity: CRITICAL**

In `~/www/twa/` there are 3 complete versions of the WebApp:
- `app.js` + `index.html` + `style.css` — the current live version
- `ns_app.js` + `ns_style.css` — an older version, ~3,000 lines
- `nsbot_twa_app.js` + `nsbot_twa_style.css` + `nsbot_twa_index.html` — another older version

**The 2 old versions are serving 0 users but polluting the server.** They should be removed.

---

### 3.2 Bot Restart Fragility (Keep-Alive Solution)
**Severity: HIGH**

The current solution (`keep_alive.php` + Alwaysdata Cron every 5 minutes) is functional but fragile:
- If the bot crashes at minute 4:59, it's down for nearly 5 minutes
- The bot process is killed by the shared host when idle
- **Proper solution:** Alwaysdata supports official "User Programs" (supervised long-running processes) in their control panel under **Advanced > Programs**. Setting up the bot there would give it automatic restart on crash with no 5-minute lag.

---

### 3.3 Inconsistent Uzbek Month Spellings
**Severity: MEDIUM**

Three different month spelling standards exist across the codebase:
- `utils.js`: `'Sentyabr'` (correct Uzbek)
- `twa_api.js`: `'Sentabr'` (shortened)
- `pdf_generator.js`: mixes both

This causes silent data-grouping bugs when months don't match string comparisons. All files need to be standardized to one spelling.

---

### 3.4 Test Files Left in Production Web Directory
**Severity: MEDIUM**

The following test files are publicly accessible on the live server:
- `https://edup.alwaysdata.net/twa/test.php`
- `https://edup.alwaysdata.net/twa/test_exec.php`
- `https://edup.alwaysdata.net/twa/test_orig.php`
- `https://edup.alwaysdata.net/twa/test_public.php`
- `https://edup.alwaysdata.net/twa/test_sock.php`
- `https://edup.alwaysdata.net/twa/test_sock2.php`
- `https://edup.alwaysdata.net/twa/test_stream.php`
- `https://edup.alwaysdata.net/twa/test_user.php`
- `https://edup.alwaysdata.net/twa/phpinfo.php`
- `https://edup.alwaysdata.net/twa/keep_alive.php` (publicly accessible = security risk, anyone can spam-trigger bot restarts)
- `https://edup.alwaysdata.net/start_bot_test.php`

These MUST be removed or password-protected immediately.

---

### 3.5 `keep_alive.php` is Publicly Accessible
**Severity: CRITICAL SECURITY**

`keep_alive.php` is accessible by anyone with the URL. Any person can:
1. Spam-call it to repeatedly restart your bot
2. Cause the bot to crash-restart in a loop
3. Cause database corruption if the bot is killed mid-write

**Fix:** Add a secret token check:
```php
if ($_GET['token'] !== getenv('KEEPALIVE_TOKEN')) { http_response_code(403); exit; }
```
And update the Alwaysdata Cron to include `?token=YOUR_SECRET`.

---

### 3.6 No Rate Limiting on API Endpoints
**Severity: MEDIUM**

`twa_api.js` has no rate limiting. A compromised Telegram account could flood the API with hundreds of finance inserts per second. Express `express-rate-limit` should be added.

---

## ✅ SECTION 4: What Is Working Well

| Feature | Status |
|---------|--------|
| Lead input (bot) | ✅ Fully working |
| Finance input — Rasmiy & Norasmiy (bot) | ✅ Fully working |
| Debtor tracking (bot + WebApp) | ✅ Fully working |
| Rejection tracking (bot + WebApp) | ✅ Fully working |
| Attendance input (bot + WebApp) | ✅ Fully working |
| Problem reporting (bot + WebApp) | ✅ Fully working |
| Empty rooms (bot + WebApp) | ✅ Fully working |
| Daily/Weekly/Monthly PDF reports | ✅ Fully working |
| General (Umumiy) report | ✅ Fully working |
| CSV export | ✅ Fully working |
| Tax report scenes (bot) | ✅ Fully working |
| Google Sheets sync | ✅ Working (every 15 min) |
| Cron system (built-in + custom) | ✅ Fully working |
| User management from WebApp | ✅ Fully working |
| WebApp auth (Telegram initData) | ✅ Secure |
| Accountability alert cron (21:00) | ✅ Working |
| WAL checkpoint cron (23:55) | ✅ Working |
| Session persistence (SQLite) | ✅ Working |
| i18n Uzbek + Russian support | ✅ Working |
| Search scene (6-month data) | ✅ Working |

---

## 🎯 Priority Fix List

| # | Issue | Effort | Impact |
|---|-------|--------|--------|
| 1 | 🔒 Delete all test PHP files from `/twa/` | 5 min | Critical |
| 2 | 🔒 Protect `keep_alive.php` with a secret token | 10 min | Critical |
| 3 | 🤖 Add `GEMINI_API_KEY` and wire AI into reports | 2 hours | High |
| 4 | 📊 Add charts to WebApp (Chart.js already loaded) | 3 hours | High |
| 5 | 🔁 Move bot to Alwaysdata "Programs" supervisor | 15 min | High |
| 6 | 🗑️ Delete 2 dead WebApp versions (`ns_app.js`, `nsbot_twa_app.js`) | 5 min | Medium |
| 7 | 📅 Fix Uzbek month spelling inconsistency (Sentyabr vs Sentabr) | 30 min | Medium |
| 8 | 📄 Add Tax Report to WebApp | 2 hours | Medium |
| 9 | 📥 Add CSV/PDF export button to WebApp | 2 hours | Medium |
| 10 | 👤 Add permission picker to `/addmanager` flow | 1 hour | Medium |
| 11 | 🔒 Add rate limiting to `twa_api.js` | 30 min | Medium |
| 12 | ✅ Add delete confirmation to Finance records | 30 min | Low |
| 13 | 🔗 Activate Notion integration | 20 min | Low |
