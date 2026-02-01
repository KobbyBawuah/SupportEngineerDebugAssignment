# Investigation Notes

> Working document to track findings during triage. Will be used to populate RUNBOOK.md, INCIDENT.md, and TICKET.md.

---

## Setup Notes

- **App runs on:** http://localhost:5000 (not 5088 as stated in README)
- **Test project issue:** Missing `using Xunit;` in `TaskApiTests.cs` — tests won't compile (setup issue, not a reported bug)

---

## Issue #1: Create task fails with 500

### Status: ✅ Reproduced & Root Cause Found

### Customer Report
> "I click Add and sometimes it just errors. If I try again a few seconds later it works."

### Reproduction Steps
1. Open http://localhost:5000/
2. Enter a task title
3. Click **Add** multiple times
4. ~35% of attempts fail with 500

**Alternative repro (Swagger):**
1. Open http://localhost:5000/swagger
2. POST to `/api/tasks` with body: `{"userId": "user-001", "title": "test"}`
3. Do NOT include `X-Client-Timestamp` header → 500 error

### Evidence

**Stack trace from terminal:**
```
System.FormatException: String '' was not recognized as a valid DateTime.
   at System.DateTime.Parse(String s)
   at SupportEngineerChallenge.Api.Endpoints.TaskEndpoints.<>c.<<MapTaskEndpoints>b__0_1>d.MoveNext() 
   in TaskEndpoints.cs:line 29
```

**API log:**
```
Request finished HTTP/1.1 POST http://localhost:5000/api/tasks - 500 0 - 23.9864ms
```

### Root Cause

**Location:** `TaskEndpoints.cs:29` + `main.js:59-66`

**API code (TaskEndpoints.cs:28-29):**
```csharp
var clientTimestamp = ctx.Request.Headers["X-Client-Timestamp"].ToString();
var createdAt = DateTime.Parse(clientTimestamp);  // Throws if empty
```

**UI code (main.js:59-66):**
```javascript
const includeHeader = Math.random() > 0.35;
if (includeHeader) {
  headers["X-Client-Timestamp"] = new Date().toISOString();
} else {
  headers["X-Client-Timestamp"] = "";  // Sends empty string ~35% of time
}
```

**Why it fails:**
1. API expects `X-Client-Timestamp` header
2. UI randomly sends empty string (~35% of requests)
3. `DateTime.Parse("")` throws `FormatException`
4. Unhandled exception → 500 error

### Proposed Fix
- **API-side:** Validate header before parsing; use `DateTime.TryParse()` or fall back to `DateTime.UtcNow` if missing/invalid
- **UI-side:** Remove random behavior, always send valid timestamp (secondary fix)

### Impact
- ~35% of task creation attempts fail
- Poor user experience
- Data loss (user thinks task was created but it wasn't)

### Fix Applied: ✅

**File changed:** `TaskEndpoints.cs:26-33`

**Before:**
```csharp
var clientTimestamp = ctx.Request.Headers["X-Client-Timestamp"].ToString();
var createdAt = DateTime.Parse(clientTimestamp);
```

**After:**
```csharp
var clientTimestamp = ctx.Request.Headers["X-Client-Timestamp"].ToString();
DateTime createdAt;
if (!DateTime.TryParse(clientTimestamp, out createdAt))
{
    logger.LogWarning("Invalid or missing X-Client-Timestamp header: '{Header}'. Using server time.", clientTimestamp);
    createdAt = DateTime.UtcNow;
}
```

**Changes made:**
1. Used `DateTime.TryParse()` instead of `DateTime.Parse()` to safely handle invalid input
2. Fall back to `DateTime.UtcNow` when header is missing/invalid
3. Added logging to track when fallback is used (helps with monitoring)
4. Injected `ILogger<Program>` for logging capability

**Tests added:** `TaskApiTests.cs`
- `CreateTask_ShouldReturn201_WhenTimestampHeaderMissing` — verifies API works without header
- `CreateTask_ShouldReturn201_WhenTimestampHeaderEmpty` — verifies API works with empty header

**What was NOT fixed (intentionally):**
- UI (`main.js`) random header behavior — the API should be defensive regardless of client behavior. The UI fix can be a follow-up ticket. This keeps the fix minimal and server-side, which is safer for production.

---

## Issue #2: Tasks list is slow for some users

### Status: ✅ Reproduced & Root Cause Found

### Customer Report
> "My task list takes a long time to load. My coworker says it's fine."

### Reproduction Steps
1. Open http://localhost:5000/swagger
2. GET `/api/tasks?userId=user-001&limit=50`
3. Check terminal logs for SQL query and response time
4. Repeat for different users (user-002, user-003, etc.)

### Evidence

**SQL Query logged (same for ALL users):**
```sql
SELECT "t"."Id", "t"."CreatedAt", "t"."Status", "t"."Title", "t"."UpdatedAt", "t"."UserId"
FROM "Tasks" AS "t"
```

**Note:** No `WHERE UserId = ...` clause. No `LIMIT` clause.

**Response times (similar for all users):**
| User | Response Times |
|------|----------------|
| user-001 | 232ms, 97ms, 107ms |
| user-002 | 217ms, 114ms, 101ms |
| user-003 | 120ms, 103ms, 101ms |
| user-004 | 101ms |
| user-005 | 123ms |

Times are similar because ALL users load ALL tasks.

### Root Cause

**Location:** `TaskEndpoints.cs:13-23`

**Code:**
```csharp
var all = await db.Tasks.AsNoTracking().ToListAsync();  // Loads ALL tasks first

var filtered = all
    .Where(t => t.UserId == userId)      // In-memory filter
    .OrderByDescending(t => t.CreatedAt)
    .Take(Math.Clamp(limit ?? 50, 1, 200))  // In-memory limit
    .ToList();
```

**Why it's slow:**
1. `ToListAsync()` fetches ALL tasks from database before any filtering
2. `Where()`, `OrderByDescending()`, `Take()` all operate in-memory AFTER loading everything
3. The `userId` and `limit` parameters are ignored at the database level
4. As dataset grows, this gets worse for ALL users (not just those with many tasks)

### Why "coworker says it's fine"
With current small dataset (~100 tasks), response times are similar (~100ms). The issue would be more visible in production with thousands of tasks.

### Proposed Fix
Move filtering to database level using LINQ-to-SQL:
```csharp
var filtered = await db.Tasks
    .AsNoTracking()
    .Where(t => t.UserId == userId)
    .OrderByDescending(t => t.CreatedAt)
    .Take(Math.Clamp(limit ?? 50, 1, 200))
    .ToListAsync();
```

### Impact
- Unnecessary database load (fetches all rows every request)
- Unnecessary memory usage (loads all tasks into memory)
- Scales poorly as data grows
- All users affected equally (not just "heavy" users)

---

## Issue #3: Duplicates / wrong order after refresh

### Status: ✅ Reproduced & Root Cause Found

### Customer Report
> "After I refresh, some tasks show up twice. Also the order looks random sometimes."

### Reproduction Steps
1. Open http://localhost:5000/
2. Note the number of rows in the task list (e.g., 50)
3. Click **Refresh** button multiple times
4. Observe: row count keeps growing, same tasks appear multiple times

### Evidence

**Before:** 50 rows, unique tasks  
**After ~15 refreshes:** 800 rows, task ID 3987 repeated many times

Each refresh triggers a GET request that returns 50 tasks, which get appended to the existing list instead of replacing it.

### Root Cause

**Location:** `main.js:46-47`

**Code:**
```javascript
state.tasks = state.tasks.concat(items)  // APPENDS instead of replacing
  .sort((a, b) => String(a.createdAt).localeCompare(String(b.createdAt)));
```

**Why it causes duplicates:**
1. `concat()` appends new items to existing `state.tasks` array
2. No deduplication — same tasks get added repeatedly
3. Each refresh multiplies the list size

**Why ordering is wrong:**
1. `String(a.createdAt).localeCompare(...)` does string comparison on dates
2. String comparison of ISO dates can give incorrect results
3. API returns newest-first, but UI re-sorts oldest-first after concat

### Proposed Fix
Replace the list instead of appending:
```javascript
state.tasks = items
  .sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt));  // Proper date sort, newest first
```

### Impact
- UI becomes unusable after a few refreshes (hundreds/thousands of duplicate rows)
- Confusing UX — users see same task multiple times
- Performance degrades as list grows
- Memory usage increases unbounded

---

## Test File Issue

**File:** `tests/SupportEngineerChallenge.Tests/TaskApiTests.cs`

**Problem:** Missing `using Xunit;` directive — tests won't compile

**Error:**
```
error CS0246: The type or namespace name 'IClassFixture<>' could not be found
error CS0246: The type or namespace name 'Fact' could not be found
```

**Fix:** Add `using Xunit;` at top of file

**Status:** ✅ Fixed — added `using Xunit;` to imports

---

## Next Steps
- [ ] Reproduce and diagnose Issue #2 (slow lists)
- [ ] Reproduce and diagnose Issue #3 (duplicates/ordering)
- [ ] Fix test file compilation
- [ ] Implement safe fixes
- [ ] Update RUNBOOK.md, INCIDENT.md, TICKET.md
