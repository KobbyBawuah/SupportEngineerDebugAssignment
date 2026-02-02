# Incident Summary

**Title:** Task Tracker — Multiple Production Issues (500 Errors, Slow Lists, UI Duplicates)  
**Date:** 2026-02-01  
**Severity:** High (Issue #1), Medium (Issue #2), Medium (Issue #3)

## Impact

### Issue #1: Create task fails with 500
- **Who:** All users creating tasks via UI
- **Impact:** ~35% of task creation attempts failed with 500 error
- **Symptoms:** Users click "Add" and see error; retry sometimes works

### Issue #2: Tasks list is slow
- **Who:** All users loading task lists
- **Impact:** Response times 265ms+ regardless of user's task count
- **Symptoms:** Sluggish page loads, poor perceived performance

### Issue #3: Duplicates after refresh
- **Who:** All users refreshing task list in UI
- **Impact:** Task list grows unbounded (50 → 800+ rows after multiple refreshes)
- **Symptoms:** Duplicate tasks appear, wrong ordering, UI becomes unusable

## Detection

- Customer reports via support tickets
- Sample API log provided showing `System.FormatException` for Issue #1
- Manual reproduction confirmed all three issues

## Timeline (UTC)

- **TBD** — Customer reports received describing intermittent 500 errors, slow lists, duplicate tasks
- **2026-02-01 ~08:00** — Investigation started
- **2026-02-01 ~08:30** — Issue #1 reproduced: 500 error on POST `/api/tasks` when `X-Client-Timestamp` header empty
- **2026-02-01 ~09:00** — Issue #2 reproduced: SQL logs show no WHERE/LIMIT, 265ms response times
- **2026-02-01 ~09:30** — Issue #3 reproduced: UI row count grows from 50 to 800+ on refresh
- **2026-02-01 ~10:00** — Root causes identified for all three issues
- **2026-02-01 ~11:00** — Fixes implemented and verified

## Root Cause

### Issue #1: Create task 500s
- **Location:** `TaskEndpoints.cs:29`
- **Cause:** `DateTime.Parse()` throws `FormatException` when `X-Client-Timestamp` header is empty
- **Contributing factor:** UI (`main.js:59-66`) randomly sends empty header ~35% of requests

```
System.FormatException: String '' was not recognized as a valid DateTime.
   at System.DateTime.Parse(String s)
   at TaskEndpoints.cs:line 29
```

### Issue #2: Slow lists
- **Location:** `TaskEndpoints.cs:13-21`
- **Cause:** `ToListAsync()` called before filtering — loads ALL tasks into memory, then filters
- **Effect:** Every request fetches entire database table regardless of userId/limit parameters

### Issue #3: Duplicates on refresh
- **Location:** `main.js:46-47`
- **Cause:** `concat()` appends new tasks to existing array instead of replacing
- **Secondary:** String comparison (`localeCompare`) used for date sorting

## Mitigation / Resolution

### Issue #1 Fix
Changed `DateTime.Parse()` to `DateTime.TryParse()` with fallback to `DateTime.UtcNow`:
```csharp
if (!DateTime.TryParse(clientTimestamp, out createdAt))
{
    logger.LogWarning("Invalid or missing X-Client-Timestamp header: '{Header}'. Using server time.", clientTimestamp);
    createdAt = DateTime.UtcNow;
}
```

### Issue #2 Fix
Moved LINQ filtering before `ToListAsync()` so database handles WHERE/ORDER BY/LIMIT:
```csharp
var filtered = await db.Tasks
    .AsNoTracking()
    .Where(t => t.UserId == userId)
    .OrderByDescending(t => t.CreatedAt)
    .Take(Math.Clamp(limit ?? 50, 1, 200))
    .ToListAsync();
```

### Issue #3 Fix
Replaced `concat()` with direct assignment and proper date sorting:
```javascript
state.tasks = items
  .sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt));
```

## Verification

### Issue #1
- POST via Swagger without header → 201 (was 500)
- POST via Swagger with empty header → 201 (was 500)
- Warning logged: `warn: Program[0] Invalid or missing X-Client-Timestamp header: ''. Using server time.`
- Added unit tests: `CreateTask_ShouldReturn201_WhenTimestampHeaderMissing`, `CreateTask_ShouldReturn201_WhenTimestampHeaderEmpty`

### Issue #2
- SQL logs now show `WHERE "UserId" = @param ... LIMIT @param`
- Response time: 265ms → 4-30ms (~10x improvement)
- Existing test `ListTasks_ShouldReturnOnlyRequestedUser` still passes

### Issue #3
- Clicked Refresh 5+ times
- Row count stayed constant at 50 (was growing to 800+)
- Order consistent (newest first)

## Follow-ups / Action Items

- [ ] Fix UI random header behavior (`main.js:59-66`) — currently sends empty `X-Client-Timestamp` ~35% of time
- [ ] Add database indexes on `UserId` and `CreatedAt` columns for further performance gains
- [ ] Add frontend test coverage for `main.js` refresh behavior
- [ ] Consider adding request validation middleware for required headers
- [ ] Review other endpoints for similar LINQ-to-Objects vs LINQ-to-SQL issues
