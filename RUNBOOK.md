# Runbook — SupportEngineerChallenge

> Update this file as part of the exercise.

## Service overview
- **Service:** SupportEngineerChallenge.Api
- **Purpose:** Minimal task tracker (create + list tasks)
- **Data store:** SQLite (`app.db` in the API working directory)

## Common commands

**Run locally**
```bash
cd src/SupportEngineerChallenge.Api
dotnet run
```

**Run tests**
```bash
dotnet test
```

## Key endpoints
- `GET /api/tasks?userId={id}&limit={n}`
- `POST /api/tasks`

## Troubleshooting checklist

### "Create task fails with 500"

**Symptoms:** Intermittent 500 errors on POST `/api/tasks`, ~35% failure rate from UI

**Error in logs:**
```
System.FormatException: String '' was not recognized as a valid DateTime.
   at System.DateTime.Parse(String s)
   at SupportEngineerChallenge.Api.Endpoints.TaskEndpoints.<>c.<<MapTaskEndpoints>b__0_1>d.MoveNext()
   in TaskEndpoints.cs:line 29
```

**Diagnosis:**
1. Check terminal logs for the above `System.FormatException`
2. Look for `X-Client-Timestamp` header in failing requests — empty or missing?
3. Test via Swagger without header to confirm reproduction

**Root cause:** `DateTime.Parse()` throws on empty/missing `X-Client-Timestamp` header

**Fix location:** `TaskEndpoints.cs:26-33` — uses `DateTime.TryParse()` with fallback to `DateTime.UtcNow`

---

### "Tasks list is slow"

**Symptoms:** All users experience ~100-265ms response times regardless of their task count

**SQL log (before fix) — no filtering:**
```sql
SELECT "t"."Id", "t"."CreatedAt", "t"."Status", "t"."Title", "t"."UpdatedAt", "t"."UserId"
FROM "Tasks" AS "t"
```
Response time: **265ms**

**SQL log (after fix) — proper filtering:**
```sql
SELECT "t"."Id", "t"."CreatedAt", "t"."Status", "t"."Title", "t"."UpdatedAt", "t"."UserId"
FROM "Tasks" AS "t"
WHERE "t"."UserId" = @__userId_0
ORDER BY "t"."CreatedAt" DESC
LIMIT @__p_1
```
Response time: **4-30ms**

**Diagnosis:**
1. Check SQL logs in terminal — look for `WHERE` and `LIMIT` clauses
2. If SQL shows `SELECT * FROM Tasks` with no filtering → problem confirmed
3. Compare response times across users (should be similar if bug present)

**Root cause:** `ToListAsync()` called before filtering — loads entire table into memory

**Fix location:** `TaskEndpoints.cs:13-21` — filtering now happens at database level

---

### "Duplicates / wrong order after refresh"

**Symptoms:** Row count grows on each refresh, same tasks appear multiple times

**Observed behavior (before fix):**
- Initial load: 50 rows
- After ~15 refreshes: 800 rows
- Same task IDs repeated many times in the list

**Diagnosis:**
1. Open browser DevTools → Network tab
2. Click Refresh, note API returns ~50 items
3. Check UI row count — if it grows each refresh, bug confirmed
4. Inspect `main.js` refresh function for `concat()` usage

**Buggy code (`main.js:46-47`):**
```javascript
state.tasks = state.tasks.concat(items)  // APPENDS instead of replacing
  .sort((a, b) => String(a.createdAt).localeCompare(String(b.createdAt)));
```

**Root cause:** UI uses `concat()` (appends) instead of replacing task list; string comparison for date sorting

**Fix location:** `main.js:46-47` — direct assignment replaces list, proper date object sorting

## Verification steps

### Issue #1: Create task (500 fix)
```bash
# Run tests
dotnet test --filter "CreateTask"
```
- POST via Swagger WITHOUT `X-Client-Timestamp` header → should return 201
- POST via Swagger WITH empty `X-Client-Timestamp: ""` → should return 201
- Check logs for warning: `warn: Program[0] Invalid or missing X-Client-Timestamp header: ''. Using server time.` Same as logs shared by the user in the sample_api_log.txt
- Create tasks from UI multiple times → no 500 errors

### Issue #2: Slow lists fix
```bash
# Run tests
dotnet test --filter "ListTasks"
```
- GET `/api/tasks?userId=user-001&limit=50` via Swagger
- Check terminal SQL logs for `WHERE "UserId" = ...` and `LIMIT`
- Response time should be <50ms (vs ~250ms before fix)

### Issue #3: Duplicates fix
- Load http://localhost:5000/
- Note row count (should be ~50)
- Click Refresh 5+ times
- Row count should stay constant (no growth)
- Order should be newest-first (by Created date)

## Rollback / mitigation

### Issue #1: Create task 500s
- **Rollback:** Revert `TaskEndpoints.cs` to use `DateTime.Parse()`
- **Mitigation:** If rollback needed, fix UI (`main.js:59-66`) to always send valid timestamp

### Issue #2: Slow lists
- **Rollback:** Revert to loading all tasks then filtering in-memory
- **Mitigation:** Add database indexes on `UserId` and `CreatedAt` columns
- **Note:** Rollback is safe but degrades performance

### Issue #3: Duplicates
- **Rollback:** Revert `main.js:46-47` to use `concat()`
- **Mitigation:** Clear `state.tasks = []` before concat as temporary fix
- **Note:** This is UI-only; no server impact
