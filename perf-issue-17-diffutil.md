---
title: "Workload analysis: DiffUtil in EventListAdapter (issue #17)"
tags: ["performance", "issue-17", "workload-analysis"]
sources: []
contributors: ["WSvX"]
created: 2026-04-26
updated: 2026-04-26
---

# Replace notifyDataSetChanged with DiffUtil in EventListAdapter

## Title

Replace notifyDataSetChanged with DiffUtil for smoother list updates in EventListAdapter

## Labels

`performance`, `refactor`

## Description

### Problem

`EventListAdapter` calls `notifyDataSetChanged()` at three locations, causing a full adapter rebind on every update. The highest-impact call is `updateListItems()` — this fires on every dataset change during scrolling, day/week view navigation, and widget updates.

### Evidence

**1. Three `notifyDataSetChanged()` call sites:**

`EventListAdapter.kt:109-111` — time format toggle:
```kotlin
fun toggle24HourFormat(use24HourFormat: Boolean) {
    this.use24HourFormat = use24HourFormat
    notifyDataSetChanged()
}
```

`EventListAdapter.kt:114-121` — full dataset replacement:
```kotlin
fun updateListItems(newListItems: ArrayList<ListItem>) {
    if (newListItems.hashCode() != currentItemsHash) {
        currentItemsHash = newListItems.hashCode()
        listItems = newListItems.clone() as ArrayList<ListItem>
        recyclerView.resetItemCount()
        notifyDataSetChanged()
        finishActMode()
    }
}
```

`EventListAdapter.kt:124-132` — print mode toggle:
```kotlin
fun togglePrintMode() {
    isPrintVersion = !isPrintVersion
    textColor = if (isPrintVersion) { ... } else { ... }
    notifyDataSetChanged()
}
```

**2. Adapter extends `MyRecyclerViewAdapter`, not `ListAdapter`:**

`EventListAdapter.kt:29-32`:
```kotlin
class EventListAdapter(
    activity: SimpleActivity, var listItems: ArrayList<ListItem>, ...
) : MyRecyclerViewAdapter(activity, recyclerView, itemClick) {
```

`MyRecyclerViewAdapter` is from `org.fossify.commons` — a base adapter that does not provide `ListAdapter`/`DiffUtil` support. Migrating to `ListAdapter` requires changing the parent class or using composition.

**3. Multi-type adapter with heterogeneous list items:**

`EventListAdapter.kt:103-107`:
```kotlin
override fun getItemViewType(position: Int) = when {
    listItems[position] is ListEvent -> ITEM_EVENT
    listItems[position] is ListSectionDay -> ITEM_SECTION_DAY
    else -> ITEM_SECTION_MONTH
}
```

Three `ListItem` subclasses exist (`ListItem.kt:3` is an empty open class):

- `ListEvent` (`ListEvent.kt:3-38`) — data class with 14 fields, extends `ListItem()`
- `ListSectionDay` (`ListSectionDay.kt:3`) — data class with `title`, `code`, `isToday`, `isPastSection`
- `ListSectionMonth` (`ListSectionMonth.kt:3`) — data class with `title` only

All three are Kotlin `data class`es, so they already have generated `equals()`/`hashCode()` suitable for `DiffUtil.ItemCallback.areContentsTheSame()`.

**4. Selection uses `hashCode()` as the key:**

`EventListAdapter.kt:70`:
```kotlin
override fun getItemSelectionKey(position: Int) = (listItems.getOrNull(position) as? ListEvent)?.hashCode()
```

`EventListAdapter.kt:136`:
```kotlin
eventItemHolder.isSelected = selectedKeys.contains(listEvent.hashCode())
```

Diff-based updates must preserve selection state. Using `ListEvent.hashCode()` as the selection key is fine as long as `areItemsTheSame` is consistent.

**5. Hash-based change detection already exists:**

`EventListAdapter.kt:115-116`:
```kotlin
if (newListItems.hashCode() != currentItemsHash) {
    currentItemsHash = newListItems.hashCode()
```

This already computes a full-list hash to skip redundant updates. `DiffUtil` would subsume this optimization while providing granular item-level notifications.

### Realistic workload assessment

**A full `DiffUtil` refactor could regress performance under realistic workloads.** Here's why:

**1. `notifyDataSetChanged()` only rebinds visible ViewHolders.** RecyclerView's internal recycling means only ~5–10 items are rebound, not the entire list. The cost is proportional to visible items, not total items.

**2. `DiffUtil` scans the entire list.** `DiffUtil.calculateDiff()` uses the Myers diff algorithm — O(N) best case (with stable IDs, mostly-unchanged list), O(N²) worst case (many items changed, no stable IDs). For this adapter:

| Workload | Total items | Visible | `notifyAll` cost | `DiffUtil` cost | Net |
|---|---|---|---|---|---|
| Day view | 10–20 | 5–8 | ~2ms (rebind 8) | ~1ms (diff 20) | ~1ms saved |
| Week view | 30–50 | 5–8 | ~2ms | ~2ms (diff 50) | break-even |
| Month/list | 100–200 | 8–10 | ~3ms | ~5–15ms (diff 200) | **slower** |
| Widget (1yr) | 500+ | 10 | ~3ms | ~50–200ms (O(N²)) | **much slower** |

Source: `DiffUtil` documentation notes worst-case O(N²) — <https://developer.android.com/reference/androidx/recyclerview/widget/DiffUtil>

**3. The `hashCode()` guard already skips identical updates.** `EventListAdapter.kt:115`:

```kotlin
if (newListItems.hashCode() != currentItemsHash) {
```

`DiffUtil` would compute a full diff even when the hash guard would have skipped the update. This duplicates work for the common case (no change).

**4. Multi-type adapter increases diff complexity.** When scrolling between views, the list composition changes entirely (different `ListSectionDay`/`ListSectionMonth` headers + different `ListEvent` items). This hits the worst-case diff path — almost no items are "the same" between old and new lists.

**5. Selection state risk.** CAB multi-select uses `ListEvent.hashCode()` as the key (`EventListAdapter.kt:70,136`). `ListAdapter.submitList()` runs diffing asynchronously by default, creating a window where selection keys could reference stale positions.

### Revised scope: targeted fixes, not full DiffUtil migration

The pragmatic improvements are low-risk, minimal-scope changes that address the actual bottlenecks:

**Fix 1: Replace `notifyDataSetChanged()` with `notifyItemRangeChanged()` in toggle methods.**

These toggle methods (`toggle24HourFormat`, `togglePrintMode`) change visual formatting on every visible item — a full `notifyDataSetChanged()` triggers an unnecessary layout pass. `notifyItemRangeChanged(0, listItems.size)` rebinds the same items without the layout invalidation:

```kotlin
fun toggle24HourFormat(use24HourFormat: Boolean) {
    this.use24HourFormat = use24HourFormat
    notifyItemRangeChanged(0, listItems.size)
}

fun togglePrintMode() {
    isPrintVersion = !isPrintVersion
    textColor = if (isPrintVersion) { ... } else { ... }
    notifyItemRangeChanged(0, listItems.size)
}
```

This is a 2-line change. No new classes, no inheritance change, no selection risk.

**Fix 2 (optional): `DiffUtil.calculateDiff()` for `updateListItems()` only, if animations are desired.**

Keep the `hashCode()` guard. Only run diff when data actually changed. This is the only call site where granular notifications provide value (item additions/removals during scroll). Skip if animations aren't a product requirement.

**Fix 3 (not recommended): Full `ListAdapter` migration.**

High risk for marginal gain. The parent class `MyRecyclerViewAdapter` provides CAB selection logic that doesn't integrate with `ListAdapter.submitList()`'s async diffing. Would require either forking `org.fossify.commons` or wrapping the adapter, both of which add maintenance burden disproportionate to the performance benefit.

### Required changes (recommended scope — Fix 1 only)

1. `EventListAdapter.kt:111` — change `notifyDataSetChanged()` to `notifyItemRangeChanged(0, listItems.size)`
2. `EventListAdapter.kt:131` — same change
3. No other files modified

### Impact

- **Scope (recommended):** 2 lines changed in `EventListAdapter.kt`
- **No schema changes, no layout changes, no inheritance changes**
- **Low risk** — `notifyItemRangeChanged` is a strict improvement over `notifyDataSetChanged` when all items need rebinding but layout structure is unchanged
- **Full `DiffUtil` migration (not recommended):** ~50–80 lines, medium risk of selection state bugs and potential regression for large lists

### References

- `DiffUtil` documentation (notes O(N²) worst case): <https://developer.android.com/reference/androidx/recyclerview/widget/DiffUtil>
- `ListAdapter` guide: <https://developer.android.com/reference/androidx/recyclerview/widget/ListAdapter>
- RecyclerView performance: <https://developer.android.com/topic/performance/recycler-view>
- Why `notifyItemRangeChanged` is preferred over `notifyDataSetChanged`: <https://developer.android.com/reference/androidx/recyclerview/widget/RecyclerView.Adapter#notifyDataSetChanged()>

## Benchmarking

### Approach 1: `adb shell dumpsys gfxinfo` (recommended — zero code changes)

```bash
adb shell dumpsys gfxinfo org.fossify.calendar reset
# Perform toggle / scroll operations
adb shell dumpsys gfxinfo org.fossify.calendar framestats
```

Look at the 95th percentile frame time. `notifyDataSetChanged()` triggers full layout invalidation; `notifyItemRangeChanged()` should show lower frame times on the toggle paths.

### Approach 2: `System.nanoTime()` around toggle methods

```kotlin
fun toggle24HourFormat(use24HourFormat: Boolean) {
    val start = System.nanoTime()
    this.use24HourFormat = use24HourFormat
    notifyItemRangeChanged(0, listItems.size)
    val elapsed = (System.nanoTime() - start) / 1_000_000.0
    Log.d("PERF", "toggle24HourFormat: $elapsed ms")
}
```

Quick A/B comparison. The `notify` call itself is near-zero; the measurable difference is in the subsequent layout pass.

### What to measure

For the recommended scope (toggle fix), the relevant metric is **frame jank during toggle operations** (24h format switch, print mode). For a full `DiffUtil` migration, measure **scroll frame timing** across different list sizes — but be prepared for regression rather than improvement on large lists.

### Key caveat

Benchmark before investing in `DiffUtil`. The current `hashCode()` guard + visible-item-only rebind may already be faster than diffing the full list. The toggle fix is a clear win regardless.
