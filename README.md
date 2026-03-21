# 🔍 NS-Bot — Professional Deep Audit Report

> **Project:** Nodir School Manager Bot (Telegram)  
> **Stack:** Node.js, Telegraf, SQLite (better-sqlite3), Google Sheets, Google Gemini AI, PDFKit  
> **Date:** 2026-03-22  
> **Codebase Analyzed:** ~8,000+ lines across 12+ core modules  

---

## 📊 Executive Summary

NS-Bot is a **Telegram-based school management system** for Nodir School (ta'lim markazi). It enables managers to submit daily reports (leads, debtors, rejections, finance, attendance, problems, empty rooms) and provides the CEO with daily/weekly/monthly analytics, PDF reports, and AI-powered insights.

The codebase is **functionally mature** with a solid architecture layer (SQLite, RBAC, i18n, graceful shutdown). However, there are **critical bugs, missing features, and production-readiness gaps** that need attention.

---

## 🚨 CRITICAL BUGS & NON-WORKING FEATURES

### 1. [fetchAllData()](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/storage.js#775-940) — Full Table Scans Every Query

> [!CAUTION]
> The [fetchAllData()](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/storage.js#775-940) function in [storage.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/storage.js) (lines 775-939) loads **ALL rows** from every table on **every single call**, then filters in JavaScript. This is a **O(n) full-scan** on every report generation.

```javascript
// PROBLEM: Fetches ALL rows then filters in JS
const leadRows = db.prepare('SELECT * FROM leads ORDER BY id').all();
for (const row of leadRows) {
    if (isWithinRange(row.timestamp)) { ... }
}
```

**Impact:** As data grows (thousands of rows per table), every daily/weekly/monthly report, AI query, and accountability check will become slower and slower. With 6 tables scanned × multiple calls per cron job, this is a serious **performance bottleneck**.

**Fix:** Use SQL `WHERE` clauses with proper timestamp filtering instead of loading all rows into memory.

---

### 2. Timestamp Format Mismatch — Date Filtering is Fragile

> [!CAUTION]
> Timestamps are stored in Uzbek locale format (`"28/02/2026, 23:12:34"`) but date filtering uses `parseUzDate()` to parse them back. This is an inherently fragile pipeline.

The [isWithinRange()](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/storage.js#790-798) function depends on `parseUzDate()` to correctly parse every timestamp. If the Uzbek locale format ever changes across Node.js versions or OS locales, **all date-based queries will silently return empty results**.

**Fix:** Store timestamps in ISO-8601 format (`YYYY-MM-DD HH:MM:SS`) and use SQL `BETWEEN` for filtering.

---

### 3. Session Persistence — Sessions Are Lost on Restart

> [!WARNING]
> The bot uses Telegraf's default **in-memory sessions** (`bot.use(session())`). Every time the bot restarts (PM2 restart, deployment, crash), **all active user sessions are lost**.

**Impact:** Any user mid-way through a wizard scene (e.g., adding a lead, filling out finance data) will have their progress silently wiped. Scene state becomes corrupted.

**Fix:** Use `@telegraf/session/sqlite` or a Redis-backed session store.

---

### 4. `ceoTaxScene` Registered But Not Accessible

> [!WARNING]
> The Tax Report scene (`ceoTaxScene`) is built in [ceo_scenes.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/ceo_scenes.js) (line 620-688) but there is **no button or menu entry** in [bot.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/bot.js) to access it. It's a dead feature.

The Reports menu only has: Daily, Weekly, Monthly, Umumiy, Export, AI — but no Tax option.

**Fix:** Add a `rep_tax` button to the Reports inline keyboard and a corresponding action handler.

---

### 5. `/addmanager` Command is a Stub

> [!WARNING]
> The `/addmanager` command (bot.js line 1214-1227) only **validates the format** and replies with a message. It does **NOT actually add the user** to the database.

```javascript
// Only sends a message, doesn't add the user
await ctx.reply(t(lang, 'users_ask_id', { id: newId }) + "\n" + t(lang, 'admin_add_manager_note'));
```

**Fix:** Implement actual user addition via `storage.addUser()` with default sections.

---

### 6. `GEMINI_API_KEY` is Listed as Required But AI Fails Silently

The [config-validator.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/config-validator.js) lists `GEMINI_API_KEY` as **required**, but [ai.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/ai.js) handles missing keys gracefully by disabling AI. This is contradictory — if the key is missing, the bot **won't start** because [validateStartup()](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/config-validator.js#95-157) returns false and calls `process.exit(1)`.

**Fix:** Move `GEMINI_API_KEY` to optional env vars (as AI is non-critical), or provide a better fallback.

---

## ⚠️ SIGNIFICANT ISSUES

### 7. No Database Backup Strategy

There is no automated backup of the SQLite database (`data/bot.db`). The sync to Google Sheets happens every 15 minutes, but it's a **destructive full-replace** — old Sheet data is cleared before writing new data. If the SQLite DB corrupts, **all data is lost**.

**Required:**
- Automated daily SQLite backups (e.g., `.backup` command or file copy)
- Keep Sheet sync as append-only or with versioning

---

### 8. Google Sheets Sync — Rate Limiting & Data Loss Risk

> [!WARNING]
> [syncAllToSheets()](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/sync.js#14-130) in [sync.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/sync.js) calls `clearSheetData()` then `appendRowsBatch()` for each table. If the bot crashes between clear and append, that sheet's data is **wiped**.

The 4-second delays between operations are a brittle workaround for Google API rate limits. The sync is 10 sequential operations × 4s = **40+ seconds** of vulnerability window.

**Fix:** Write to a new sheet range first, then swap (atomic replacement pattern).

---

### 9. No Rate Limiting for User Input

The bot has no protection against rapid-fire message flooding. A malicious or mis-configured user could:
- Send hundreds of "add lead" requests, filling the DB
- Trigger hundreds of expensive [fetchAllData()](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/storage.js#775-940) calls
- Exhaust the Gemini API quota via `/ask` spam

**Fix:** Add per-user rate limiting middleware (e.g., 1 request per second, 10 per minute).

---

### 10. Notion Integration is One-Way and Partial

[notion.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/notion.js) only syncs **rejections** to Notion (`addRejection()`). No other data type is synced, and there's no reverse sync. This means the Notion database only has a partial view.

---

### 11. [warningsDb.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/warningsDb.js) — Separate State Without Schema

The lead monitor cron uses `warningsDb` to track last warning times, but this module is separate from the main [db.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/db.js) schema. It likely uses its own file or in-memory state.

---

## 📋 MISSING FEATURES (Should Be Implemented)

### A. Data Editing & Correction

| Feature | Status | Priority |
|---|---|---|
| Edit existing lead entries | ❌ Missing | 🔴 High |
| Edit finance entries | ❌ Missing | 🔴 High |
| Edit debtor entries | ❌ Missing | 🔴 High |
| Edit attendance records | ❌ Missing | 🟡 Medium |
| Undo last action | ❌ Missing | 🟡 Medium |
| View edit history/audit log | ❌ Missing (only qazdorlar_log exists) | 🟡 Medium |

Currently, users can only **add** and **delete** data. There is no way to correct a mistake in an existing record — you must delete it and re-add, which loses metadata.

---

### B. Daily Report PDF Auto-Send

| Feature | Status | Priority |
|---|---|---|
| Auto-send daily PDF to CEO at 09:00 | ❌ Missing | 🔴 High |

The `morning_summary` cron job sends a **text-only** daily report. There is no automatic PDF delivery — only a manual `/sendpdf` command. Weekly and monthly crons DO send PDFs automatically.

---

### C. Branch/Filial Management

| Feature | Status | Priority |
|---|---|---|
| Add/edit/delete branches via bot | ❌ Missing | 🟡 Medium |
| Branch-specific reports | ❌ Missing | 🟡 Medium |
| Branch-level performance comparison | ❌ Missing | 🟡 Medium |

Empty rooms and problems reference branches by free-text string. There's no master list of branches, making data inconsistent (e.g., "1-filial" vs "1 filial" vs "Filial 1").

---

### D. Search Functionality

| Feature | Status | Priority |
|---|---|---|
| Search for specific leads/debtors/entries | Partially built ([search_scene.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/search_scene.js) exists) | 🟡 Medium |
| Full-text search across all data | ❌ Missing | 🟡 Medium |

The [search_scene.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/search_scene.js) exists but needs review of what it actually searches.

---

### E. Dashboard / Summary Command

| Feature | Status | Priority |
|---|---|---|
| Quick `/dashboard` or `/summary` command | ❌ Missing | 🟡 Medium |

Managers currently must navigate through menus to see any data. A quick `/dashboard` command showing today's key metrics at a glance would be very useful.

---

### F. Data Export Formats

| Feature | Status | Priority |
|---|---|---|
| CSV export | ✅ Exists | — |
| Excel (.xlsx) export | ❌ Missing | 🟢 Low |
| JSON export for API integration | ❌ Missing | 🟢 Low |

---

### G. Notification System Enhancements

| Feature | Status | Priority |
|---|---|---|
| Custom reminder to specific users | ✅ Exists (custom crons) | — |
| Deadline/due date reminders | ❌ Missing | 🟡 Medium |
| Payment due notifications | ❌ Missing | 🟡 Medium |
| Low-attendance alerts | ❌ Missing | 🟡 Medium |

The lead monitor sends alerts when leads > 15, but there are no similar monitors for:
- Attendance dropping below threshold
- Finance balance going negative
- Debtor amounts exceeding limits
- Problems unresolved for X days

---

### H. Reporting & Analytics Gaps

| Feature | Status | Priority |
|---|---|---|
| Year-over-year comparison | ❌ Missing | 🟡 Medium |
| Trend graphs (visual) | ❌ Missing | 🟡 Medium |
| Conversion rate (leads → enrolled) | ❌ Missing | 🔴 High |
| Student enrollment tracking | ❌ Missing | 🔴 High |
| Revenue per student metric | ❌ Missing | 🟡 Medium |

---

### I. Security & Compliance

| Feature | Status | Priority |
|---|---|---|
| Activity/audit logging | ❌ Missing (only qarzdorlar_log exists) | 🔴 High |
| Login/access logs | ❌ Missing | 🟡 Medium |
| Two-factor for CEO commands | ❌ Missing | 🟡 Medium |
| Data retention policy | ❌ Missing | 🟡 Medium |
| GDPR/data export for users | ❌ Missing | 🟢 Low |

---

### J. Testing

| Feature | Status | Priority |
|---|---|---|
| Config validator tests | ✅ Exists | — |
| Cleanup utils tests | ✅ Exists | — |
| Logger tests | ✅ Exists | — |
| Shutdown handler tests | ✅ Exists | — |
| **Storage layer tests** | ❌ Missing | 🔴 High |
| **Scene/flow tests** | ❌ Missing | 🔴 High |
| **Report generation tests** | ❌ Missing | 🔴 High |
| **Integration tests** | ❌ Missing | 🟡 Medium |

Only 4 unit tests exist for utility modules. **Zero tests** for the core business logic (storage, scenes, reports, cron).

---

## 🏗️ ARCHITECTURAL IMPROVEMENTS NEEDED

### 1. Deployed Version Gap

The **deployed version** (`ns-bot-audit` on the server) has only the **old package.json** (no SQLite, no jest, no tests), while the **v6 backup** has the full evolution. Confirm which version is actually running on the server.

### 2. Code Organization

- [bot.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/bot.js) at **1,391 lines** is a monolith. Action handlers, menu builders, and middleware should be extracted.
- [storage.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/storage.js) at **1,191 lines** conflates data access, business logic, and migration code.
- [ceo_reports.js](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/ceo_reports.js) at **927 lines** has heavy report formatting mixed with data fetching.

### 3. Environment Configuration

- No [.env.example](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/.env.example) in the deployed version
- `CEO_TELEGRAM_ID` is used in env but not consistently — some cron jobs broadcast to all `reports` users instead
- `pm2` is a production dependency in the audit version but a devDependency in v6

### 4. Error Recovery

- No retry logic for Google Sheets API calls
- No retry logic for Telegram API sends (only catches and logs)
- No dead-letter queue for failed cron job sends

---

## ✅ WHAT WORKS WELL

| Feature | Quality |
|---|---|
| RBAC with section-based permissions | ⭐⭐⭐⭐ Solid |
| Bilingual i18n (Uzbek + Russian) | ⭐⭐⭐⭐ Comprehensive |
| SQLite schema design | ⭐⭐⭐⭐ Well-indexed |
| Graceful shutdown handling | ⭐⭐⭐⭐ Professional |
| Config validation on startup | ⭐⭐⭐⭐ Good |
| Input sanitization | ⭐⭐⭐ Adequate |
| Cron job management (toggle, override, custom) | ⭐⭐⭐⭐ Flexible |
| PDF report generation | ⭐⭐⭐ Functional |
| Scene-stuck escape middleware | ⭐⭐⭐⭐ Well-designed |
| Norasmiy → Qarzdorlar auto-deduction | ⭐⭐⭐⭐ Smart business logic |
| Potential income calculation from empty rooms | ⭐⭐⭐⭐ Innovative |
| AI anomaly detection & Q&A | ⭐⭐⭐ Good when API key present |
| [editMessageText](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/bot.js#95-106) error suppression | ⭐⭐⭐⭐ Clean |

---

## 🎯 RECOMMENDED PRIORITY ACTIONS

### Immediate (Critical)
1. **Fix [fetchAllData()](file:///d:/new_projects/asad/ns-bot-v6%20=%20backup/storage.js#775-940)** — Replace full table scans with SQL WHERE filters
2. **Add persistent sessions** — Use SQLite-backed sessions to survive restarts
3. **Add daily PDF auto-send** to the morning cron job
4. **Wire up `ceoTaxScene`** — Add button in Reports menu
5. **Verify which version is running on the server** — Deploy v6 if the old version is live

### Short-term (Important)
6. Add data editing capabilities (edit existing entries)
7. Implement database backup cron (daily SQLite backup)
8. Add rate limiting middleware
9. Build branch management (master branch list)
10. Add activity/audit logging for all operations

### Medium-term (Enhancement)
11. Write unit tests for storage.js, scenes.js, ceo_reports.js
12. Add conversion rate tracking (leads → enrolled students)
13. Implement threshold-based alerts (attendance, finance, debtor limits)
14. Extract bot.js into modular route handlers
15. Store timestamps in ISO-8601 format
