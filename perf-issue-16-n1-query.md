---
title: "Workload analysis: N+1 query in EventsHelper (issue #16)"
tags: ["performance", "issue-16", "workload-analysis"]
sources: []
contributors: ["WSvX"]
created: 2026-04-26
updated: 2026-04-26
---

# Fix N+1 query in EventsHelper.getEventsSync

## Title

Fix N+1 query in EventsHelper when decorating birthday/anniversary events

## Labels

`bug`, `performance`

## Description

### Problem

`EventsHelper.kt` calls `eventsDB.getEventWithId()` inside a per-event loop during event decoration. This is an N+1 query problem that fires once for every event in the result set.

### Evidence

**1. The loop that triggers N queries:**

`EventsHelper.kt:565-567` — iterates over every event calling `decorateEvent()`:

```kotlin
events.forEach {
    decorateEvent(it, birthDayEventId, anniversaryEventId, calendarColors)
}
```

**2. The per-event DB call:**

`EventsHelper.kt:583` — inside `decorateEvent()`, a single-row query runs for each event:

```kotlin
val originalEvent = eventsDB.getEventWithId(event.id!!)
```

**3. The DAO method being called N times:**

`EventsDao.kt:37-38`:

```kotlin
@Query("SELECT * FROM events WHERE id = :id AND type = $TYPE_EVENT")
fun getEventWithId(id: Long): Event?
```

Note: This query filters by `TYPE_EVENT` only, so it returns `null` for tasks. The result (`originalEvent`) is only used when the event is a birthday or anniversary (`EventsHelper.kt:584-586`). For all other events this query is wasted entirely.

**4. The widget hot path:**

`EventListWidgetAdapter.kt:199` — every widget refresh calls `getEventsSync()`:

```kotlin
context.eventsHelper.getEventsSync(fromTS, toTS, applyTypeFilter = true, callback = {
```

`EventsHelper.kt:453` — also called from the main activity event loading:

```kotlin
getEventsSync(fromTS, toTS, eventId, applyTypeFilter, searchQuery, overrideCalendarIds, callback)
```

**5. No batching alternative exists:**

`EventsDao.kt` has no `IN (:ids)` variant for bulk lookups. The closest existing methods are all single-ID (`getEventWithId`, `getTaskWithId`, `getEventOrTaskWithId` at lines 38, 41, 44).

### Current behavior

For N events returned by a query, the method makes N+1 database calls (1 for the initial batch query + N individual `getEventWithId` lookups in `decorateEvent`). For events that are neither birthdays nor anniversaries, the query result is fetched but never used — pure waste.

### Expected behavior

Batch all event ID lookups into a single query and pass a `Map<Long, Event>` into `decorateEvent()`.

### Suggested fix

1. Add a DAO method to `EventsDao`:

```kotlin
@Query("SELECT * FROM events WHERE id IN (:ids)")
fun getEventsWithIds(ids: List<Long>): List<Event>
```

2. In `getEventsSync()` (before line 565), pre-fetch originals:

```kotlin
val originalEvents = eventsDB.getEventsWithIds(events.mapNotNull { it.id })
    .associateBy { it.id!! }
```

3. Change `decorateEvent()` signature to accept `originalEvents: Map<Long, Event>` and replace `EventsHelper.kt:583` with:

```kotlin
val originalEvent = originalEvents[event.id!!]
```

### Optimization opportunity

The lookup at `EventsHelper.kt:583` is only needed when `isBirthday || isAnniversary` (line 586). The check for `birthDayEventId` and `anniversaryEventId` can be done **before** the batch query — if neither exists, skip the query entirely:

```kotlin
val needOriginals = birthDayEventId != -1L || anniversaryEventId != -1L
val originalEvents = if (needOriginals) {
    eventsDB.getEventsWithIds(events.mapNotNull { it.id }).associateBy { it.id!! }
} else {
    emptyMap()
}
```

This avoids the query for the majority of users who have no local birthday/anniversary calendars.

### Realistic workload assessment

**This fix is unlikely to be user-perceptible.** Here's why:

**1. Primary key lookups are fast.** The query at `EventsDao.kt:37-38` (`WHERE id = :id`) is a B-tree lookup in SQLite. These complete in ~0.05–0.2ms each. Room also caches prepared statements and SQLite's page cache makes repeated PK lookups effectively RAM reads.

| Workload | Events | N queries | Est. total time | Fix saves |
|---|---|---|---|---|
| Widget (1-day view) | 5–15 | 5–15 | 0.5–3ms | ~2ms |
| Widget (1-month) | 30–80 | 30–80 | 3–16ms | ~12ms |
| Power user (1000+ events) | 200+ | 200+ | 20–40ms | ~30ms |

**2. Runs on a background thread.** `EventsHelper.kt:452` wraps the call in `ensureBackgroundThread { }`. Widget updates have a 10-second ANR window. A 16ms query cost is negligible in this context.

**3. Result is used only for birthdays/anniversaries.** Lines 584–586 guard the actual usage — most events are neither. For users without local birthday/anniversary calendars, every query returns a result that's never used.

### Revised scope: skip-when-not-needed is the real win

The highest-value change is **not** the batching — it's the early-exit that skips the query entirely when no birthday/anniversary calendars exist:

```kotlin
val needOriginals = birthDayEventId != -1L || anniversaryEventId != -1L
if (!needOriginals) {
    // Skip all per-event lookups — no birthday/anniversary calendars configured
    events.forEach { event.color = calendarColors.get(event.calendarId) ?: ... }
    callback(events)
    return
}
```

This eliminates N queries for the majority of users. For the minority who do have birthday/anniversary calendars, the batched `IN (:ids)` query is still correct hygiene.

### Impact

- **Scope:** ~15 lines changed in `EventsHelper.kt`, ~3 lines in `EventsDao.kt`
- **No schema changes, no UI changes**
- **User-perceptible impact:** unlikely for typical users. The fix is code hygiene + avoiding unnecessary work.
- **Low risk** — logic is self-contained and behavior-identical

### References

- Android SQLite performance best practices: <https://developer.android.com/topic/performance/sqlite>
- Room DAO query documentation: <https://developer.android.com/training/data-storage/room/accessing-data>

## Benchmarking

### Approach 1: Room query logging (recommended)

Enable Room query logging in the debug build to count actual SQL statements:

```kotlin
// In the RoomDatabase builder
builder.setQueryCallback({ sqlQuery, bindArgs ->
    Log.d("RoomSQL", "Query: $sqlQuery | Args: $bindArgs")
}, Executors.newSingleThreadExecutor())
```

Count `SELECT * FROM events WHERE id =` occurrences per `getEventsSync()` call. Before fix: N calls. After fix: 0 (no birthday calendars) or 1 (batched).

### Approach 2: Manual timing

Add temporary instrumentation to `getEventsSync()`:

```kotlin
val startTime = System.nanoTime()
// ... existing code ...
val elapsed = (System.nanoTime() - startTime) / 1_000_000.0
android.util.Log.d("PERF", "getEventsSync: $elapsed ms for ${events.size} events")
```

Test with calendars of varying sizes (50, 200, 1000 events). Compare before/after the fix.

### What to measure

The meaningful metric is **query count**, not wall-clock time. Given that each PK lookup is ~0.1ms and runs off the UI thread, timing differences will be within noise. Query count (N → 0 or 1) is the clear before/after signal.
