# Time Logger Skill

Scope: workspace

Version: 0.1

Summary
-------
This skill adds a new entry to `time-log.md` at the project root whenever the user logs time, keeps the running total accurate, and reminds them to commit and push. The expectation is 30–40 hours total across the project.

When to use
-----------
- "Log [N] hours [description]"
- "I spent [N] hours on [description] today"
- "Add time: [description], [N] hours, [optional date]"
- "Show my time log total"
- "Auto-log my agent time" / "How much time have I spent in Copilot?"
- "Log my session time since [date]"

Log file location
-----------------
`time-log.md` at the root of the project (same level as `app/`). Create it if it does not exist.

Log file format
---------------
The file uses a markdown table. Always keep the header row, one data row per session, and a bold Total row at the bottom. Never merge sessions — each work session is its own row.

```markdown
# Project Time Log

| Date | What | Hours |
|---|---|---|
| 3/27/2026 | Worked on initial designs | 2.5 |
| 3/29/2026 | Set up tech stack | 2 |
| **Total** | | **4.5** |
```

Workflow (step by step)
-----------------------
1. Read the current `time-log.md` to get the existing rows and current total.
2. Determine the date:
   - Use the date the user provides if given.
   - Otherwise use today's date in M/D/YYYY format (no leading zeros).
3. Append a new data row above the Total row.
4. Recompute the total by summing all Hours values (treat blank/non-numeric as 0).
5. Update the **Total** row with the new sum.
6. Write the updated file.
7. Remind the user to commit and push:
   > "Don't forget to commit and push your time log entry!"
   > Suggested commit message: `"Time log: [description] ([N] hrs)"`

Auto-track from VS Code Copilot sessions
-----------------------------------------
When the user asks to auto-log agent/session time, use the `session_store_sql` tool to calculate active working time from turn timestamps, then follow the normal workflow above.

**How active time is calculated:**
- Time is estimated from gaps between consecutive turns within a Copilot session.
- Gaps ≤ 30 minutes are counted as active working time; gaps > 30 minutes are treated as breaks and excluded.
- A fixed 5-minute buffer is added per working day to account for reading the final response.
- This is an estimate — always show the user the breakdown and ask for confirmation before logging.

**SQL query to run** (using `session_store_sql`):

```sql
WITH gaps AS (
    SELECT
        date(t.timestamp) AS work_date,
        (julianday(t.timestamp) - julianday(
            LAG(t.timestamp) OVER (PARTITION BY t.session_id ORDER BY t.turn_index)
        )) * 1440 AS gap_minutes
    FROM turns t
    JOIN sessions s ON s.id = t.session_id
    WHERE s.cwd LIKE '%AndroidStudioProjects%Clue%'
)
SELECT
    work_date,
    ROUND(SUM(CASE WHEN gap_minutes <= 30 THEN gap_minutes ELSE 0 END) / 60.0 + 0.08, 2) AS hours
FROM gaps
WHERE work_date IS NOT NULL
GROUP BY work_date
ORDER BY work_date
```

- To limit to sessions since a specific date, add: `AND date(t.timestamp) >= 'YYYY-MM-DD'`
- To exclude dates already in the time log, check the last logged date first and add that filter.

**After running the query:**
1. Show the user a table of dates and estimated hours.
2. Ask: "Would you like to log these as individual entries (one per day) or combined as one entry?"
3. For each date/entry confirmed, run the normal log workflow.
4. Default description for auto-logged entries: `"Copilot agent session"` — ask user if they want a more specific description.

Decision points
---------------
- If no date is given: use today's date.
- If the file does not exist: create it with the header and the new entry.
- If the user asks to see the total: read the file and report the Total row value and hours remaining toward 40.
- If the total exceeds 40 hours: note it but still log the entry.
- If auto-tracking: run the SQL query, show results, confirm before writing.

Quality checks
--------------
- Hours must be a positive number (decimals allowed, e.g. 1.5, 0.5).
- Description must be non-empty.
- Total row must always be the last row in the table.
- The Total hours must equal the sum of all individual rows.

Progress toward goal
--------------------
After every log entry, report:
- Hours logged so far
- Hours remaining to reach 40
- Percentage complete

Example output after logging
-----------------------------
> Logged 2 hours: "Implemented game board grid UI"
> Time log updated. Running total: 6.5 / 40 hours (16% complete).
> 33.5 hours remaining.
> Don't forget to commit and push: `git add time-log.md && git commit -m "Time log: Implemented game board grid UI (2 hrs)" && git push`

Edit history
------------
- 2026-10-14 5:16 PM — Initial draft created.
- 2026-10-14 5:30 PM — Added auto-tracking from VS Code Copilot session history using turn-gap analysis.
