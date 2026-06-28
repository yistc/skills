# iOS SwiftUI Entry List Performance Playbook

Use this reference when optimizing iOS SwiftUI entry lists, RSS/feed reader lists, large `List` rows, context menu latency, or row image/favicon jank.

## Operating Rules

- Treat user reports and prior investigation as hypotheses. Read the actual code path before accepting the proposed fix.
- Keep `List` by default on iOS when it provides navigation, swipe actions, context menus, and platform behavior. Replacing it with `LazyVStack` is a larger architecture change and is not the first move.
- Optimize row hot paths before changing list architecture: row `body`, context menu construction, swipe action construction, image decoding, formatter/string work, and broad observable reads.
- Preserve interaction semantics. It is acceptable to precompute menu availability, but the action itself should still compute exact IDs/state at click time when precision matters.
- If a project says not to run iOS Simulator, do not run it. Use unit tests, generic iOS build, and user device verification.

## Diagnosis Checklist

1. Inspect the iOS list implementation:
   ```sh
   rg -n 'List \\{|ForEach\\(|contextMenu|swipeActions|NavigationLink' .
   ```
2. Look for O(n) work inside row modifiers or menu builders:
   ```sh
   rg -n 'firstIndex|filter|map|prefix|suffix|sorted|contains\\(' <list files>
   ```
3. Look for synchronous image work in row `body`:
   ```sh
   rg -n 'UIImage\\(|NSImage\\(|Image\\(uiImage|Image\\(nsImage|CGImageSource|Data\\?' .
   ```
4. Check whether repeated blobs are carried per row, such as favicon data repeated for every entry from the same feed.
5. Verify identity is stable. Prefer stable domain IDs such as `serverID`, not `\.self`.

## Preferred Fix Patterns

### Context menu availability

Do not scan the full visible entry array from each row's `.contextMenu` disabled state.

Preferred pattern:

- Build a small value type from the current visible entries when the list projection changes.
- Precompute sets such as `serverIDsWithUnreadAbove` and `serverIDsWithUnreadBelow` in O(n).
- In the menu builder, use O(1) lookup:
  ```swift
  availability.hasUnreadEntries(anchoredAt: entry.serverID, direction: .above) == false
  ```
- Keep the actual action path unchanged or exact:
  ```swift
  let serverIDs = entryServerIDsToMarkRead(in: entries, anchoredAt: entry.serverID, direction: direction)
  ```

Avoid a single `@State` cache like `markReadAboveIDs`/`markReadBelowIDs` unless it is keyed by the current anchor entry. Above/below results depend on the row being pressed.

### Row image/favicon decoding

Do not decode `UIImage(data:)`, `NSImage(data:)`, or ImageIO thumbnails directly in row `body`.

Preferred pattern:

- Deduplicate image data by stable owner ID, such as `feedID`.
- Decode/downsample off the main actor using ImageIO:
  ```swift
  CGImageSourceCreateThumbnailAtIndex(
    source,
    0,
    [
      kCGImageSourceCreateThumbnailFromImageAlways: true,
      kCGImageSourceCreateThumbnailWithTransform: true,
      kCGImageSourceShouldCacheImmediately: true,
      kCGImageSourceThumbnailMaxPixelSize: 64,
    ] as CFDictionary
  )
  ```
- Store decoded images in list-level state keyed by feed ID.
- Pass decoded images into row views. Rows should only render an existing image or a lightweight placeholder.
- Remove stale cache entries when the visible list changes; discard cancelled or obsolete decode results.

### Projection-layer derived state

For data derived from the visible list, prefer computing alongside the list projection instead of inside SwiftUI row builders.

Good candidates:

- mark-above / mark-below availability
- row display flags
- stable lightweight row metadata
- preformatted strings only if profiling or code review shows formatter cost

Do not use `@State` as a generic cache without a clear invalidation key such as `entriesRevision`, selected feed, filter, or sorted visible entry IDs.

## Anti-patterns

- Replacing `List` with `LazyVStack` before proving `List` itself is the bottleneck.
- Adding pagination to hide a hot-path bug when product semantics require all visible entries.
- Putting `entries.filter`, `entries.firstIndex`, `entryServerIDsToMarkRead`, or image decoding in row `body` / `.contextMenu` builders.
- Blanket `.equatable()` on rows without proving equality is cheaper than rebuilding and inputs are value-semantic.
- Trusting a performance report's proposed patch when its cache key does not match the behavior being cached.

## Testing and Verification

- Add Swift Testing unit tests for pure derived helpers:
  - unread above anchor enables above menu
  - unread below anchor enables below menu
  - no unread on side disables that side
  - missing anchor disables both sides
- Preserve existing tests for exact action output, such as IDs returned by mark-above / mark-below helpers.
- Run a macOS test target if shared code is covered there.
- Run iOS generic build:
  ```sh
  xcodebuild build -project <Project>.xcodeproj -scheme <iOSScheme> -destination 'generic/platform=iOS'
  ```
- Device verification should cover:
  - 1000+ entry scrolling
  - long-press context menu open latency
  - mark above/below enabled state
  - favicon placeholders resolving without wrong images after feed/filter changes

## Known Good Outcome From RSS App KIS-314

In an iOS RSS entry list with `List`, `NavigationLink`, row context menus, swipe actions, and feed favicons:

- The proposed report correctly identified O(n) context menu disabled checks, but its global `@State` cache proposal was wrong because above/below results depend on the current row anchor.
- A better fix precomputed `EntryReadRangeAvailability` with O(n) work per list projection and O(1) menu lookup.
- Scrolling jank was more likely caused by synchronous row favicon decoding than by text view count. Rows had repeated `faviconData` for entries from the same feed and decoded image data in `body`.
- A list-level async favicon cache keyed by `feedID`, using ImageIO downsampling and placeholders, made the list feel much smoother on device.
- Verification used Swift Testing, macOS shared tests, iOS generic build, and user device testing; no iOS Simulator was run because the project forbade it.
