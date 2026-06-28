# macOS Large List + Memory Playbook

Use this reference when optimizing macOS apps with thousands of rows, RSS/feed/log/mail-style lists, SwiftUI `List`/`Table` jank, SwiftData `@Query` hangs, CPU spikes, or memory targets such as normal physical footprint under 300 MB.

## Operating Rules

- Treat list UI and data loading as one system. Replacing SwiftUI `List` is not enough if the data source still materializes every full model object.
- Optimize for physical footprint and responsiveness, not `ps RSS` alone. Prefer `sample <pid> 1` and `vmmap -summary <pid>` for memory evidence.
- Keep user data semantics intact. Do not cap sync or hide entries just to reduce memory unless product requirements explicitly allow pagination or limits.
- Verify with the real store size when available. Synthetic small fixtures do not prove large-list behavior.

## Diagnosis Checklist

1. Count data first:
   ```sh
   sqlite3 "$HOME/Library/Application Support/<AppName>/db.sqlite" \
     "select count(*) total, sum(ZSTATUS='unread') unread, sum(ZSTATUS='read') read from ZENTRY"
   ```
2. Find hot full-fetch paths:
   ```sh
   rg -n '@Query|List\\(|Table\\(|ForEach\\(|modelContext.fetch|propertiesToFetch|htmlContent|reloadData' .
   ```
3. Check whether list state stores:
   - SwiftData model arrays
   - full HTML/body/content fields
   - image/blob data for all rows
   - derived arrays rebuilt in SwiftUI `body`
4. Measure before and after:
   ```sh
   pgrep -n -f '<AppName>.app/Contents/MacOS/<AppName>'
   ps -p <pid> -o pid,pcpu,pmem,rss,vsz,command
   sample <pid> 1 -file /tmp/app.sample.txt >/dev/null 2>&1
   vmmap -summary <pid> > /tmp/app.vmmap.txt
   rg -n 'Physical footprint|TOTAL|MALLOC|SQLite page cache|mapped file' /tmp/app.sample.txt /tmp/app.vmmap.txt
   ```

## Preferred macOS List Architecture

For large single-column macOS lists, use `NSTableView` via `NSViewRepresentable`.

Required table properties:

- view-based table with custom `NSTableCellView`
- constant `rowHeight`
- no SwiftUI row hosting views for hot rows
- `makeView(withIdentifier:owner:)` reuse
- efficient `numberOfRows(in:)`
- selection synchronized manually
- `reloadData()` only when a revision/token changes, not on every SwiftUI update

Use SwiftUI `List` only for small/moderate lists or non-hot platforms. `LazyVStack` delays creation but does not provide AppKit-style reuse.

## Data Source Rules

Avoid holding full persistent models in list state. Use lightweight sendable snapshots:

- row ID/server ID
- title
- author/source
- date
- status/starred flags
- small metadata such as reading time

Do not include:

- full HTML/content/body
- relationship arrays
- large blobs/images unless strictly needed for visible rows

Load detail content lazily by selected ID.

## SwiftData Rules

- Avoid `@Query` for large macOS lists. It can materialize many model instances and trigger expensive SwiftUI invalidation.
- `FetchDescriptor.propertiesToFetch` can help, but do not rely on it as the only memory boundary for very large datasets.
- Push filtering and sorting into the store whenever possible.
- Add indexes for columns used in predicates and sort orders:
  - server ID lookups
  - status + published date
  - feed ID + status + published date
  - starred + published date
- If SwiftData indexes do not materialize in an existing SQLite store, a bounded manual `CREATE INDEX IF NOT EXISTS` repair may be acceptable:
  - open the known app store URL
  - set `sqlite3_busy_timeout`
  - run inside `BEGIN IMMEDIATE` / `COMMIT`
  - rollback and log warnings on failure
  - treat SwiftData internal table names as a risk and keep failures nonfatal

## SQLite Projection Pattern

When SwiftData list fetches still load too much, add a read-only SQLite projection path for list snapshots while keeping SwiftData for writes, migrations, and detail fetches.

Pattern:

1. Open the SwiftData SQLite store read-only:
   ```swift
   sqlite3_open_v2(storeURL.path, &db, SQLITE_OPEN_READONLY | SQLITE_OPEN_NOMUTEX, nil)
   sqlite3_busy_timeout(db, 5_000)
   ```
2. Select only list columns:
   ```sql
   SELECT ZSERVERID, ZFEEDID, ZTITLE, ZAUTHOR, ZPUBLISHEDAT, ZSTATUS,
     ZSTARRED, ZREADINGTIME, ZFEEDTITLE
   FROM ZENTRY
   WHERE ZSTATUS = ?
   ORDER BY ZPUBLISHEDAT DESC
   ```
3. Do not select full content columns such as `ZHTMLCONTENT`.
4. Bind parameters. Never interpolate user search text into SQL.
5. Escape `LIKE` patterns:
   ```sql
   ZTITLE LIKE ? ESCAPE '\' COLLATE NOCASE
   ```
6. Keep an in-memory SwiftData fallback for tests using `isStoredInMemoryOnly`.

Use this only when the project accepts coupling to SwiftData's SQLite table layout. Document the coupling in code review/final notes.

## Search and Badges

- Do search in SQL where practical, not by fetching all entries and filtering in Swift.
- Badge counts should be aggregate queries (`GROUP BY feedID`) or trusted server counters, not full row materialization.
- If unread counts are already stored on feeds and no global search is active, use feed unread counters directly.

## Sync Rules

- Do not keep an old artificial entry cap if the product expects all unread entries.
- Fetch large remote entry sets in pages.
- Upsert database writes in batches and skip no-op saves.
- Avoid per-entry fetch/save loops:
  - batch-fetch existing feeds/entries by server IDs
  - compare DTO to model before writing
  - save once per meaningful batch
- For read entries, incremental sync using `changed_after` is acceptable only after local read count is known to be complete enough; otherwise do a full backfill to heal old caps.

## AppKit Cell Rules

- Precompute display strings where repeated formatting shows up in profiles.
- Use static `DateFormatter` or cached formatted dates.
- Keep row subviews fixed and update `stringValue`, `image`, hidden flags.
- Cap or trim favicon/image caches. Decode images only for rows/feeds that can appear.
- Do not call layout recursion APIs from inside `layout`; call `super.layout()` and assign frames directly for hot cells.

## Verification Gates

Before marking done:

- `xcodebuild build` succeeds.
- Relevant tests pass; add tests for pagination, sync limits, projection snapshots, and search escaping when touched.
- Real data measurements are recorded:
  - total row count
  - visible/filter row count
  - physical footprint current and peak
  - CPU after idle
- Confirm the app opened the real SQLite store, not fallback in-memory:
  ```sh
  lsof -p <pid> | rg 'Application Support/.*/db.sqlite'
  ```

## Known Good Outcome From RSS App

In a real RSS app with 48,666 entries and 5,503 unread:

- macOS sidebar and article list moved from SwiftUI list views to `NSTableView`.
- SwiftUI `@Query` full-fetch list path was removed.
- list snapshot uses lightweight rows and read-only SQLite projection, excluding `htmlContent`.
- selected article detail loads full HTML lazily.
- unread sync cap was removed so 5k+ unread entries show.
- measured physical footprint stabilized around 108.7 MB with peak around 146.0 MB, CPU around 0.9%.

Use these numbers as a sanity benchmark, not a guarantee for other apps.
