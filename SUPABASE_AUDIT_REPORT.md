# LASUASA Voting Portal — Supabase Audit Report

**Date:** 2026-07-01  
**Project:** shambakiuro-sketch/Lasuasa-voting-portal-  
**File Audited:** `index.html`

---

## EXECUTIVE SUMMARY

✅ **Credentials:** Hardcoded (production risk)  
✅ **Project URL:** Valid Supabase project URL detected  
✅ **Anon Key:** Valid JWT detected  
✅ **Tables:** 4 tables configured in SQL schema  
⚠️ **Logging:** Minimal — no request instrumentation  
⚠️ **Error Handling:** Basic — catches errors but doesn't log details  

---

## 1. CREDENTIALS AUDIT

### ✅ Location Found
- **File:** `index.html`, lines 24–26
- **Format:** Hardcoded constants (NOT environment variables)

### Credential Details
```javascript
const SUPABASE_URL = "https://snvhlgnjzvdqvqbdrhyh.supabase.co";
const SUPABASE_ANON_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
const ADMIN_PASSWORD = "Themisvanguard";
```

### 🚨 SECURITY ISSUES
1. **Hardcoded credentials in public file** — CRITICAL RISK
   - Anon key is publicly visible in the HTML source
   - Admin password is hardcoded (easy to brute-force)
   - If repo is public, credentials are exposed to the internet

2. **No environment variable support**
   - Credentials cannot be rotated without code change
   - Same credentials deployed to all environments

### RECOMMENDATION
- Move credentials to environment variables or `.env` file
- Use a build tool (Vite, Webpack, or similar) to inject secrets at build time
- Regenerate the Anon Key in Supabase (current key is compromised if repo is public)
- Store admin password securely (hash it, or use a separate admin auth table)

---

## 2. SUPABASE CLIENT AUDIT

### ✅ Client Implementation
- **Location:** Lines 35–101
- **Type:** Custom REST client (no official SDK)
- **Methods:** `query`, `insert`, `update`, `remove`, `subscribe`

### ✅ Connection Details
```javascript
const base = "https://snvhlgnjzvdqvqbdrhyh.supabase.co";
const rest = (path) => `${base}/rest/v1/${path}`;
```

**Endpoints used:**
- `GET /rest/v1/{table}` — fetch data
- `POST /rest/v1/{table}` — insert
- `PATCH /rest/v1/{table}?{filters}` — update
- `DELETE /rest/v1/{table}?{filters}` — delete
- `WebSocket wss://.../realtime/v1/websocket` — real-time updates

### ✅ Headers Configured
```javascript
const headers = {
  "Content-Type": "application/json",
  "apikey": key,
  "Authorization": `Bearer ${key}`
};
```

**Status:** ✅ Correct. Both `apikey` and `Authorization` headers ensure compatibility with Supabase.

---

## 3. DATABASE TABLES AUDIT

### Expected Schema (from lines 224–267)
```sql
CREATE TABLE election_config (
  id TEXT PRIMARY KEY DEFAULT 'main',
  voting_open BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE posts (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE candidates (
  id TEXT PRIMARY KEY,
  post_id TEXT NOT NULL REFERENCES posts(id),
  name TEXT NOT NULL,
  photo TEXT  -- base64 data URL
);

CREATE TABLE voters (
  matric_no TEXT PRIMARY KEY,
  full_name TEXT NOT NULL,
  level TEXT NOT NULL,
  code TEXT NOT NULL,
  voted BOOLEAN DEFAULT false,
  votes JSONB DEFAULT '{}'
);
```

### 3.1 Table Queries by Function

| **Function** | **Line** | **Operation** | **Table** | **Columns** | **Status** |
|---|---|---|---|---|---|
| `useElectionData()` | 331 | Query | `election_config` | `id`, `voting_open` | ✅ |
| `useElectionData()` | 332 | Query | `posts` | `id`, `title`, `created_at` | ✅ |
| `useElectionData()` | 333 | Query | `candidates` | All | ✅ |
| `useElectionData()` | 334 | Query | `voters` | All | ✅ |
| `toggleVoting()` | 453 | Update | `election_config` | `voting_open` | ✅ |
| `addPost()` | 459 | Insert | `posts` | `id`, `title` | ✅ |
| `removePost()` | 465 | Delete | `candidates` | `post_id` (WHERE) | ✅ |
| `removePost()` | 466 | Delete | `posts` | `id` (WHERE) | ✅ |
| `addCandidate()` | 475 | Insert | `candidates` | `id`, `post_id`, `name`, `photo` | ✅ |
| `removeCandidate()` | 480 | Delete | `candidates` | `id` (WHERE) | ✅ |
| `addVoter()` | 489 | Insert | `voters` | All columns | ✅ |
| `bulkImport()` | 499 | Insert | `voters` | All columns | ✅ |
| `resetVoter()` | 504 | Update | `voters` | `voted`, `votes` | ✅ |
| `removeVoter()` | 505 | Delete | `voters` | `matric_no` (WHERE) | ✅ |
| `submitVote()` | 886 | Update | `voters` | `voted`, `votes` | ✅ |

### ✅ COLUMN COVERAGE
All columns queried **exist** in the schema.

### ⚠️ Row-Level Security (RLS)
```sql
CREATE POLICY "allow all" ON {table} FOR ALL USING (true) WITH CHECK (true);
```

**Status:** ⚠️ CRITICAL ISSUE
- RLS is enabled but policy allows **any authenticated user** (including anon) to:
  - Read all voter data (GDPR violation risk)
  - Modify any voter's record (vote tampering risk)
  - Delete election data

**Recommendation:** Implement stricter RLS policies:
```sql
-- Voters can only read/update their own record
CREATE POLICY "voters_own_record" ON voters
  FOR SELECT USING (auth.uid()::text = matric_no);

CREATE POLICY "voters_update_own" ON voters
  FOR UPDATE USING (auth.uid()::text = matric_no)
  WITH CHECK (auth.uid()::text = matric_no);

-- Candidates/posts are read-only for voters
CREATE POLICY "posts_read_only" ON posts FOR SELECT USING (true);
CREATE POLICY "candidates_read_only" ON candidates FOR SELECT USING (true);

-- Admin-only modifications (requires admin auth)
CREATE POLICY "admin_manage_posts" ON posts
  FOR INSERT USING (auth.jwt() ->> 'role' = 'admin');
```

---

## 4. ERROR HANDLING AUDIT

### Current Implementation (Lines 44, 53, 63, 70)
```javascript
if (!res.ok) throw new Error(await res.text());
```

**Status:** ❌ INSUFFICIENT
- Errors are swallowed in catch blocks (lines 337–338, 448)
- No request logging
- No error context (which table, which operation, what parameters)

### Example Failure Flow
1. **Request fails** → `res.ok` is false
2. **Error thrown** → Generic message from Supabase API
3. **Catch block** → Only `e.message` is stored
4. **UI shows** → "Error: {message}" (user sees raw Supabase error)
5. **No logs** → Developer cannot diagnose root cause

### Failure Scenarios Not Instrumented

| Scenario | Likely Error | Logged? |
|---|---|---|
| Wrong project URL | ENOTFOUND / 404 | ❌ |
| Wrong anon key | "Invalid JWT" / 401 | ❌ |
| Missing table | "relation not found" / 404 | ❌ |
| Missing column | "column does not exist" / 400 | ❌ |
| RLS policy blocks read | "new row violates policy" / 403 | ❌ |
| Network timeout | TIMEOUT / ECONNABORTED | ❌ |
| Invalid JSONB in votes | "invalid input" / 400 | ❌ |

---

## 5. IMPROVED ERROR HANDLING WITH LOGGING

### Add to `index.html` (after line 101)

```javascript
// ── Supabase Request Logger ────────────────────────────────────────
const LOG_ENABLED = true; // Set to false in production
const LOGS = [];

function logRequest(operation, table, details, status, errorObj) {
  const timestamp = new Date().toISOString();
  const log = {
    timestamp,
    operation,  // "query", "insert", "update", "delete", "subscribe"
    table,
    details,    // { match, data, params, etc. }
    status,     // "pending", "success", "error"
    error: errorObj ? {
      message: errorObj.message,
      code: errorObj.code,
      statusCode: errorObj.statusCode,
      details: errorObj.details,
      hint: errorObj.hint,
      stack: errorObj.stack?.split('\n')[0]
    } : null
  };
  LOGS.push(log);
  if (LOG_ENABLED) console.table(log);
  return log;
}

function getSupabaseErrorType(errorText) {
  if (errorText.includes("relation") && errorText.includes("not found")) return "MISSING_TABLE";
  if (errorText.includes("column") && errorText.includes("does not exist")) return "MISSING_COLUMN";
  if (errorText.includes("violates")) return "RLS_POLICY_BLOCK";
  if (errorText.includes("Invalid JWT")) return "INVALID_ANON_KEY";
  if (errorText.includes("ENOTFOUND")) return "WRONG_PROJECT_URL";
  if (errorText.includes("TIMEOUT")) return "NETWORK_TIMEOUT";
  if (errorText.includes("invalid input")) return "INVALID_DATA_TYPE";
  return "UNKNOWN";
}

// Enhanced Supabase client with logging
function createClient(url, key) {
  const headers = { "Content-Type": "application/json", "apikey": key, "Authorization": `Bearer ${key}` };
  const base = url.replace(/\/+$/, "");
  const rest = (path) => `${base}/rest/v1/${path}`;

  async function query(table, params = {}) {
    logRequest("query", table, { params }, "pending");
    try {
      const qs = Object.entries(params).map(([k, v]) => `${k}=${encodeURIComponent(v)}`).join("&");
      const res = await fetch(`${rest(table)}${qs ? "?" + qs : ""}`, {
        headers: { ...headers, "Prefer": "return=representation" }
      });
      if (!res.ok) {
        const errText = await res.text();
        const errorType = getSupabaseErrorType(errText);
        logRequest("query", table, { params, statusCode: res.status }, "error", {
          message: errText,
          code: errorType,
          statusCode: res.status
        });
        throw new Error(`[${errorType}] ${errText}`);
      }
      const data = await res.json();
      logRequest("query", table, { params, rowsReturned: data.length }, "success");
      return data;
    } catch (e) {
      if (!e.message.includes("[")) {
        logRequest("query", table, { params }, "error", e);
      }
      throw e;
    }
  }

  async function insert(table, data) {
    logRequest("insert", table, { data }, "pending");
    try {
      const res = await fetch(rest(table), {
        method: "POST",
        headers: { ...headers, "Prefer": "return=representation" },
        body: JSON.stringify(data),
      });
      if (!res.ok) {
        const errText = await res.text();
        const errorType = getSupabaseErrorType(errText);
        logRequest("insert", table, { data }, "error", {
          message: errText,
          code: errorType,
          statusCode: res.status
        });
        throw new Error(`[${errorType}] ${errText}`);
      }
      const result = await res.json();
      logRequest("insert", table, { data, rowsInserted: result.length }, "success");
      return result;
    } catch (e) {
      if (!e.message.includes("[")) {
        logRequest("insert", table, { data }, "error", e);
      }
      throw e;
    }
  }

  async function update(table, match, data) {
    logRequest("update", table, { match, data }, "pending");
    try {
      const qs = Object.entries(match).map(([k, v]) => `${k}=eq.${encodeURIComponent(v)}`).join("&");
      const res = await fetch(`${rest(table)}?${qs}`, {
        method: "PATCH",
        headers: { ...headers, "Prefer": "return=representation" },
        body: JSON.stringify(data),
      });
      if (!res.ok) {
        const errText = await res.text();
        const errorType = getSupabaseErrorType(errText);
        logRequest("update", table, { match, data }, "error", {
          message: errText,
          code: errorType,
          statusCode: res.status
        });
        throw new Error(`[${errorType}] ${errText}`);
      }
      const result = await res.json();
      logRequest("update", table, { match, data, rowsUpdated: result.length }, "success");
      return result;
    } catch (e) {
      if (!e.message.includes("[")) {
        logRequest("update", table, { match, data }, "error", e);
      }
      throw e;
    }
  }

  async function remove(table, match) {
    logRequest("delete", table, { match }, "pending");
    try {
      const qs = Object.entries(match).map(([k, v]) => `${k}=eq.${encodeURIComponent(v)}`).join("&");
      const res = await fetch(`${rest(table)}?${qs}`, { method: "DELETE", headers });
      if (!res.ok) {
        const errText = await res.text();
        const errorType = getSupabaseErrorType(errText);
        logRequest("delete", table, { match }, "error", {
          message: errText,
          code: errorType,
          statusCode: res.status
        });
        throw new Error(`[${errorType}] ${errText}`);
      }
      logRequest("delete", table, { match }, "success");
    } catch (e) {
      if (!e.message.includes("[")) {
        logRequest("delete", table, { match }, "error", e);
      }
      throw e;
    }
  }

  function subscribe(table, callback) {
    logRequest("subscribe", table, {}, "pending");
    const wsUrl = `${base.replace("https://", "wss://").replace("http://", "ws://")}/realtime/v1/websocket?vsn=1.0.0&apikey=${key}`;
    let ws;
    let reconnectTimer;

    function connect() {
      try {
        ws = new WebSocket(wsUrl);
        ws.onopen = () => {
          logRequest("subscribe", table, { status: "connected" }, "success");
          ws.send(JSON.stringify({
            topic: "realtime:*",
            event: "phx_join",
            payload: {
              config: {
                broadcast: { self: true },
                presence: {},
                postgres_changes: [{ event: "*", schema: "public", table }]
              }
            },
            ref: 0
          }));
        };
        ws.onmessage = (e) => {
          const msg = JSON.parse(e.data);
          if (msg.event === "postgres_changes" || (msg.payload && msg.payload.data)) {
            logRequest("subscribe", table, { event: msg.event }, "success");
            callback();
          }
          if (msg.event === "heartbeat") {
            ws.send(JSON.stringify({ topic: "phoenix", event: "heartbeat", payload: {}, ref: null }));
          }
        };
        ws.onclose = () => {
          logRequest("subscribe", table, { status: "disconnected" }, "pending");
          reconnectTimer = setTimeout(connect, 3000);
        };
        ws.onerror = (err) => {
          logRequest("subscribe", table, { status: "error" }, "error", err);
          ws.close();
        };
      } catch (e) {
        logRequest("subscribe", table, {}, "error", e);
      }
    }

    connect();
    return () => {
      clearTimeout(reconnectTimer);
      if (ws) ws.close();
    };
  }

  return { query, insert, update, remove, subscribe, getLogs: () => LOGS };
}
```

---

## 6. VERIFICATION CHECKLIST

### Pre-Deployment
- [ ] **Run SQL schema** in Supabase SQL Editor
- [ ] **Enable RLS policies** (use restrictive policies, not "allow all")
- [ ] **Enable Realtime replication** on `posts`, `candidates`, `voters`, `election_config`
- [ ] **Test connection:**
  ```javascript
  const test = await db.query("election_config");
  console.log("✅ Connected:", test);
  ```
- [ ] **Check CORS settings** in Supabase (if running on different domain)

### Post-Deployment
- [ ] Open browser console (F12 → Console tab)
- [ ] Navigate to admin page
- [ ] Try adding a post
- [ ] Check browser console for logs
- [ ] Verify `LOGS` array contains successful operations
- [ ] Test error scenario (e.g., disconnect internet, try adding voter)
- [ ] Check console for error code (e.g., "NETWORK_TIMEOUT")

---

## 7. COMMON FAILURE MODES & DIAGNOSTICS

### Scenario 1: "MISSING_TABLE"
**Symptom:** Admin dashboard shows "Error: relation [...] not found"  
**Cause:** SQL schema not run in Supabase  
**Fix:**
1. Open Supabase SQL Editor
2. Paste the SQL from Setup Guide (lines 224–267)
3. Click "Run"
4. Refresh the voting portal

### Scenario 2: "INVALID_ANON_KEY"
**Symptom:** "Error: Invalid JWT"  
**Cause:** Wrong anon key in code  
**Fix:**
1. Go to Supabase Settings → API
2. Copy the "anon / public" key (starts with `eyJ`)
3. Update `SUPABASE_ANON_KEY` on line 25

### Scenario 3: "WRONG_PROJECT_URL"
**Symptom:** "Error: ENOTFOUND" or "Error: 404"  
**Cause:** Typo in project URL  
**Fix:**
1. Go to Supabase Settings → API
2. Copy "Project URL" (e.g., `https://abcxyz.supabase.co`)
3. Update `SUPABASE_URL` on line 24
4. **Do NOT add `/rest/v1/` to the URL**

### Scenario 4: "RLS_POLICY_BLOCK"
**Symptom:** "Error: new row violates policy"  
**Cause:** RLS policy denies the operation  
**Fix:** Check RLS policies in Supabase:
1. Go to Database → RLS
2. Click each table and verify policies
3. For development, use `FOR ALL USING (true)` (not production-safe)
4. For production, implement auth-based policies

### Scenario 5: "INVALID_DATA_TYPE"
**Symptom:** "Error: invalid input syntax for type json"  
**Cause:** `votes` column expects valid JSON  
**Example issue:** Sending `votes: undefined` instead of `votes: {}`  
**Fix:** Ensure all data matches schema before insert:
```javascript
// ❌ WRONG
await db.insert("voters", { ..., votes: undefined });

// ✅ CORRECT
await db.insert("voters", { ..., votes: {} });
```

---

## 8. PRODUCTION HARDENING

### Recommended Changes
1. **Move credentials to environment:**
   ```javascript
   const SUPABASE_URL = process.env.VITE_SUPABASE_URL;
   const SUPABASE_ANON_KEY = process.env.VITE_SUPABASE_ANON_KEY;
   const ADMIN_PASSWORD = process.env.VITE_ADMIN_PASSWORD;
   ```

2. **Disable logging in production:**
   ```javascript
   const LOG_ENABLED = process.env.NODE_ENV !== "production";
   ```

3. **Add rate limiting to admin operations:**
   ```javascript
   const opCount = {};
   function rateLimit(operation, limit = 5, windowMs = 60000) {
     const key = operation;
     if (!opCount[key]) opCount[key] = [];
     opCount[key] = opCount[key].filter(t => Date.now() - t < windowMs);
     if (opCount[key].length >= limit) throw new Error("Rate limit exceeded");
     opCount[key].push(Date.now());
   }
   ```

4. **Implement vote integrity checks:**
   ```javascript
   function validateVotes(votes, posts) {
     for (const [postId, candidateId] of Object.entries(votes)) {
       if (!posts.find(p => p.id === postId)) throw new Error("Invalid post");
       // Verify candidateId exists in that post
     }
   }
   ```

---

## 9. SUMMARY TABLE

| Issue | Severity | Line(s) | Fix |
|---|---|---|---|
| Hardcoded credentials | 🔴 CRITICAL | 24–26 | Move to `.env` file |
| Weak RLS policies | 🔴 CRITICAL | 264–267 | Implement auth-based policies |
| No request logging | 🟡 HIGH | 44, 53, 63, 70 | Add instrumentation (see section 5) |
| Poor error messages | 🟡 HIGH | 337–338, 448 | Log full error objects |
| No data validation | 🟠 MEDIUM | 489, 499 | Add schema validation |
| No rate limiting | 🟠 MEDIUM | 453, 458, 484 | Add rate limit checks |
| Hardcoded admin password | 🟡 HIGH | 26 | Hash or move to table |

---

## 10. RECOMMENDED NEXT STEPS

1. **Immediate (Before Launch):**
   - [ ] Move credentials to environment variables
   - [ ] Implement restrictive RLS policies
   - [ ] Add enhanced logging from section 5
   - [ ] Regenerate the exposed anon key

2. **Short-term (Week 1):**
   - [ ] Add input validation for all voter data
   - [ ] Implement admin authentication (not just password)
   - [ ] Add rate limiting to prevent brute-force attacks

3. **Medium-term (Month 1):**
   - [ ] Set up Supabase audit logs (Settings → Logs)
   - [ ] Implement vote tamper detection (checksums, signatures)
   - [ ] Add backup/recovery procedures

4. **Long-term (Ongoing):**
   - [ ] Monitor Supabase usage and costs
   - [ ] Review RLS policies monthly
   - [ ] Update dependencies (React 18 is stable but monitor for security patches)

---

**Report Generated:** 2026-07-01  
**Auditor:** GitHub Copilot  
**Status:** ⚠️ REQUIRES ATTENTION BEFORE PRODUCTION
