# iOS SwiftUI WidgetKit

Use this reference when implementing or diagnosing iOS WidgetKit widgets, especially SwiftUI widget extensions that display app data from SwiftData, SQLite, RSS/feed apps, or any local cache.

## Mental model

A widget is not a small always-running app. It is a separate extension process with tight time, memory, and scheduling limits. WidgetKit asks the extension for rendered timeline entries, archives those views, and later displays the archived result.

Default stance:

- The containing app owns sync, persistence, authentication, and business state.
- The widget extension should display a small, precomputed snapshot.
- Do not make the widget responsible for full server sync, large database scans, or app-like refresh behavior.
- Design for delayed, skipped, or coalesced refreshes.

For an RSS app, treat the widget as "latest snapshot from the last successful app sync", not as an independent RSS client.

## Three widget states

Keep these states separate:

- `placeholder(in:)`: synchronous, instant, generic representation. No disk, network, database, or heavy image decoding.
- `getSnapshot(in:completion:)`: transient preview, commonly used by Widget Gallery. If `context.isPreview` is true, return representative sample data quickly.
- `getTimeline(in:completion:)`: Home Screen / actual widget data. Return a small number of entries and call completion exactly once.

Gallery preview working does not prove Home Screen timeline works. Gallery often exercises `getSnapshot`; Home Screen exercises `getTimeline`.

If Gallery shows real-looking data but Home Screen is redacted/skeleton, suspect timeline generation, extension crash, timeout, memory pressure, signing/App Group access, or stale WidgetKit cache.

## Static vs AppIntent configuration

Use `StaticConfiguration` for widgets with no user-editable parameters.

Use `AppIntentConfiguration` only when the widget has real configuration parameters or interactive App Intent needs.

Avoid empty `WidgetConfigurationIntent` types just to satisfy a template. They add an extra metadata/configuration path and have historically been involved in confusing placeholder behavior. For simple widgets, static configuration is simpler and more reliable.

## Shared data and App Groups

The app and widget extension have separate sandboxes. Use an App Group when the widget needs app-prepared data.

Both targets must have the same App Group entitlement:

```swift
FileManager.default.containerURL(
  forSecurityApplicationGroupIdentifier: "group.example.app"
)
```

The physical container path is system-managed. Do not hardcode it. When the app and its extensions are uninstalled, the container is normally removed unless another installed app from the same developer still uses that App Group.

For App Store/TestFlight issues, inspect the signed app and appex entitlements. Xcode project entitlements are not enough if provisioning profiles do not include the App Group.

## Preferred data flow

Prefer this flow:

1. Main app syncs and updates its normal database.
2. Main app materializes a small widget-specific snapshot.
3. Main app writes the snapshot into the App Group container.
4. Main app calls `WidgetCenter.shared.reloadTimelines(ofKind:)` or `reloadAllTimelines()`.
5. Widget timeline reads the small snapshot and renders.

For example:

```swift
struct WidgetSnapshot: Codable, Sendable {
  var unreadCount: Int
  var articles: [WidgetArticle]
}

struct WidgetSnapshotCache: Sendable {
  let url: URL

  func load() -> WidgetSnapshot {
    do {
      let data = try Data(contentsOf: url)
      return try JSONDecoder().decode(WidgetSnapshot.self, from: data)
    } catch {
      return WidgetSnapshot(unreadCount: 0, articles: [])
    }
  }

  func save(_ snapshot: WidgetSnapshot) throws {
    try FileManager.default.createDirectory(
      at: url.deletingLastPathComponent(),
      withIntermediateDirectories: true
    )
    let data = try JSONEncoder().encode(snapshot)
    try data.write(to: url, options: .atomic)
  }
}
```

Keep the cached snapshot display-ready:

- Store only fields needed by the widget.
- Limit list counts by widget family.
- Store display IDs, titles, source names, dates, and small image data or image references.
- Avoid giving the widget a full ORM object graph.

## SQLite and SwiftData in widgets

Opening the app's SwiftData/Core Data/SQLite store directly from a widget is fragile.

Risks:

- Timeline generation may exceed WidgetKit's time budget.
- The extension may exceed memory budget.
- App and widget can contend for SQLite locks.
- App Group SQLite access can trigger intermittent `SQLITE_BUSY`, disk I/O errors, or `0xDEAD10CC` style lock terminations.
- Fetching full lists just to display two rows wastes the widget budget.

If a direct SQLite fallback is unavoidable:

- Open read-only.
- Use a short busy timeout.
- Query only the exact fields needed.
- Use `COUNT(*)` for counts.
- Use `LIMIT` for article rows.
- Avoid constructing full app list snapshots.
- Catch all errors and return a harmless empty widget state.

The robust path is still: main app prepares a small JSON/plist/UserDefaults snapshot and the widget reads that.

## Refresh behavior

Widget refresh is not real-time and not guaranteed.

Timeline policy:

- `.after(date)` means WidgetKit may ask for a new timeline after that date. It is not a timer guarantee.
- `.atEnd` asks for a new timeline after the last entry date.
- `.never` means the widget relies on explicit reloads from the app or push.

`WidgetCenter.shared.reloadAllTimelines()` and `reloadTimelines(ofKind:)` request a reload, but the system may delay, coalesce, or skip reloads based on power, usage, and budgets.

If the app must sync with a server to discover new data, the widget will not know about new data unless one of these happens:

- The user opens the app and it syncs.
- Background App Refresh / BGTask wakes the app and it syncs.
- A push notification wakes the app or widget infrastructure.
- The widget itself performs a lightweight network request, which should be treated as a last resort and still must fail gracefully.

For RSS apps without push or reliable background sync, explain clearly that the widget shows the most recent app-prepared snapshot.

## Common skeleton / redacted placeholder causes

When the Home Screen widget stays skeleton/redacted:

- `getTimeline` is too slow.
- `getTimeline` never calls completion.
- Completion is called more than once.
- The extension crashes while building the timeline or rendering.
- Widget view decodes huge images or loads too much data.
- App Group entitlement is missing from the app or extension's signed profile.
- The widget reads a missing or corrupt shared file and does not provide a fallback.
- Data Protection / privacy-sensitive behavior causes placeholder display while locked.
- WidgetKit is displaying stale cached content from a previous install/build.
- The widget family receives too much content and layout breaks.

Debug sequence:

1. Confirm Gallery preview vs Home Screen behavior separately.
2. Confirm `placeholder`, `getSnapshot`, and `getTimeline` return different appropriate entries.
3. Replace timeline with a hardcoded entry. If Home Screen works, data loading is the problem.
4. Replace data loading with a tiny App Group JSON read. If it works, database/network work was too heavy.
5. Inspect signed entitlements for app and `.appex`.
6. Check Console/Xcode device logs for the widget extension process.
7. Delete old widgets and re-add after installing a changed widget kind/layout.
8. If necessary, delete the app, restart the device, reinstall, and add the widget again.

## Image handling

Images are a common widget memory failure source.

Rules:

- Do not decode original-size images when a 16-64 px icon is enough.
- Prefer ImageIO thumbnail creation before `UIImage(data:)`.
- Strip or avoid large blobs in cached widget snapshots.
- Use SF Symbols or small local assets as fallback.

Example:

```swift
func widgetImage(from data: Data?) -> UIImage? {
  guard let data,
    let source = CGImageSourceCreateWithData(data as CFData, nil),
    let image = CGImageSourceCreateThumbnailAtIndex(
      source,
      0,
      [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: 64,
      ] as CFDictionary
    )
  else {
    return nil
  }

  return UIImage(cgImage: image)
}
```

## Layout and families

Each widget family needs its own content density.

Do not show the same number of rows in `.systemMedium` and `.systemLarge`.

Practical defaults for article widgets:

- `.systemSmall`: count or one simple metric.
- `.systemMedium`: header + 1-2 compact rows.
- `.systemLarge`: header + 4-6 rows.

For medium widgets:

- Use explicit padding.
- Use smaller fonts and tighter spacing.
- Keep titles to one line when metadata is also shown.
- Limit metadata to one line.
- Add `.minimumScaleFactor` to counts and titles.
- Set `frame(maxWidth:maxHeight:alignment:)` on the root container.
- Clip or reduce content rather than letting text overflow rounded widget bounds.

The provider should limit entries by family, and the view should defensively truncate again:

```swift
private func maxArticleCount(for family: WidgetFamily) -> Int {
  switch family {
  case .systemLarge, .systemExtraLarge:
    return 6
  case .systemMedium:
    return 2
  default:
    return 1
  }
}
```

Test with long Chinese titles, large unread counts, Dynamic Type, tinted Home Screen appearance, dark mode, and stale/no-data states.

## Widget identity and cache

Widget `kind` strings are persistent identity. Changing the kind can force the system to treat it as a new widget type, but also breaks existing installed widgets.

When changing layout or provider behavior:

- Reinstall the app.
- Remove old Home Screen widgets.
- Add the widget again.
- Do not assume an existing widget instance immediately adopts new timeline behavior.

Use stable `kind` values for shipped widgets unless intentionally migrating.

## Verification

For this user's repos:

- Do not run iOS Simulator when project docs say real-device testing only.
- Use generic iOS build for compile verification:

```bash
xcodebuild build -project <Project>.xcodeproj -scheme <iOSScheme> -destination 'generic/platform=iOS' CODE_SIGNING_ALLOWED=NO
```

- Use focused Swift Testing tests for snapshot serialization/cache logic.
- Ask the user to verify on device when Home Screen WidgetKit behavior is the core risk.

Good tests:

- Missing cache returns empty snapshot.
- Cache save/load round-trips.
- SQLite fallback returns unread count and recent articles with a limit.
- Large article lists are truncated before reaching the widget.

## Decision checklist

Before implementing a widget, answer:

- Does the widget have user configuration? If no, use `StaticConfiguration`.
- What is the smallest snapshot the widget needs?
- Who writes that snapshot, and when?
- What does the widget show when snapshot is missing?
- How many rows fit in each family?
- What triggers reloads?
- Is delayed refresh acceptable for the product?
- Can images be thumbnails or symbols?
- How will App Group entitlements be verified in signed builds?

Prefer boring, small, precomputed data over clever timeline logic.
