# macOS SwiftUI Settings Window Management

Use this reference when a macOS SwiftUI Settings scene, `openSettings`, menu bar app preferences window, or AppKit-hosted settings window has problems with focus, frontmost ordering, restored position, first-open centering, or visible position jumps.

## Core Rules

- Treat placement restoration and frontmost ordering as separate problems.
- Do not show the Settings window at an old restored position and then call `center()` after it appears. Users can see the old position flash before the window jumps.
- Prefer Scene-level SwiftUI APIs when the deployment target supports them. For macOS 14-compatible apps, use AppKit frame autosave cleanup before opening the window.
- Keep any focus retry bounded. Never add an infinite retry loop around window activation.
- Always verify the actual deployment target before using macOS 15+ SwiftUI scene modifiers.

## Position Restoration

SwiftUI/AppKit may restore Settings window frames through AppKit's window frame autosave system. The data is stored in the app's user defaults under keys shaped like:

```text
NSWindow Frame <autosave-name>
```

Common Settings autosave names include:

```swift
"KindowSettingsWindow" // app-specific identifier, if the app sets one
"com_apple_SwiftUI_Settings_window" // observed SwiftUI Settings identifier
```

These saved frames are system/AppKit persistence, not necessarily custom app code. If a Settings window appears off-center after relaunch, first inspect AppKit/SwiftUI autosave before adding your own position memory.

For macOS 14-compatible code that should ignore prior Settings positions, clear known autosaves before `openSettings()` creates the window:

```swift
@MainActor
private enum SettingsWindowFocus {
  static let swiftUISettingsWindowIdentifier = "com_apple_SwiftUI_Settings_window"
  static let frameAutosaveNames = [
    "KindowSettingsWindow",
    swiftUISettingsWindowIdentifier
  ]
}

@MainActor
private func clearSettingsWindowFrameAutosaves() {
  for autosaveName in SettingsWindowFocus.frameAutosaveNames {
    NSWindow.removeFrame(usingName: autosaveName)
  }
}
```

Call this before `openSettings()`, not after the window becomes visible. `NSWindow.removeFrame(usingName:)` removes frame data stored under that name from application user defaults.

If the app can set a stable identifier on the actual Settings window, do it. This improves focus lookup and gives the app a known autosave name to clear:

```swift
struct SettingsWindowAccessor: NSViewRepresentable {
  func makeNSView(context: Context) -> NSView {
    let view = NSView()
    DispatchQueue.main.async {
      view.window?.identifier = NSUserInterfaceItemIdentifier("KindowSettingsWindow")
    }
    return view
  }

  func updateNSView(_ nsView: NSView, context: Context) {}
}
```

Use it as a small background/accessory view inside the Settings root. Keep this AppKit escape narrow.

## macOS 15+ Preferred Placement

When the deployment target is macOS 15 or newer, prefer SwiftUI scene modifiers for declarative behavior:

```swift
Settings {
  SettingsView()
}
.restorationBehavior(.disabled)
.defaultWindowPlacement { content, context in
  let displayBounds = context.defaultDisplay.visibleRect
  let size = content.sizeThatFits(.unspecified)
  let position = CGPoint(
    x: displayBounds.midX - size.width / 2,
    y: displayBounds.midY - size.height / 2
  )
  return WindowPlacement(position: position, size: size)
}
```

Availability:

- `Scene.restorationBehavior(_:)`: macOS 15.0+
- `Scene.defaultWindowPlacement(_:)`: macOS 15.0+

If the project still builds for macOS 14, do not reference these modifiers unguarded. They will fail compilation. Either keep the AppKit autosave cleanup path or split scene declarations with availability-safe code that the compiler accepts for the app's minimum target.

## Dynamic Settings Tab Sizing

When Settings tabs have very different natural sizes, it is normal for the Settings window to grow or shrink while the user switches tabs. Do not treat this as an AppKit frame-management problem first.

Preferred approach on macOS 15+:

```swift
Settings {
  SettingsView()
}
.windowResizability(.contentSize)
.windowIdealSize(.fitToContent)
```

Inside the Settings root, express the selected tab's intended content size through SwiftUI layout:

```swift
struct SettingsView: View {
  @AppStorage("selectedSettingsTab") private var selectedSettingsTab = SettingsTab.general
  @State private var displayContentSize = SettingsWindowSizing.displayPreferredSize

  var body: some View {
    TabView(selection: $selectedSettingsTab) {
      GeneralSettingsView()
      DisplaySettingsView(displayContentSize: displayContentSize)
      ShortcutSettingsView()
    }
    .frame(
      width: settingsContentSize?.width,
      height: settingsContentSize?.height
    )
  }

  private var settingsContentSize: CGSize? {
    switch selectedSettingsTab {
      case .display:
        displayContentSize
      case .shortcut:
        SettingsWindowSizing.shortcutContentSize
      default:
        nil
    }
  }
}
```

This lets SwiftUI and the Settings scene negotiate the window size from content. Avoid calling `window.setFrame(...)` during tab selection changes. Repeated manual frame fitting can create visible drift because AppKit frame origins, titlebar/chrome deltas, autosave restoration, animation, and SwiftUI layout updates are all trying to own the same window at once.

If a tab size depends on the current display, calculate content size from the window's `screen?.visibleFrame`, subtract the current chrome delta, clamp to a minimum and maximum, then write only the SwiftUI content size state:

```swift
@MainActor
static func displayContentSize(for window: NSWindow) -> CGSize {
  guard let visibleFrame = window.screen?.visibleFrame ?? NSScreen.main?.visibleFrame else {
    return displayPreferredSize
  }

  let chrome = CGSize(
    width: max(0, window.frame.width - window.contentLayoutRect.width),
    height: max(0, window.frame.height - window.contentLayoutRect.height)
  )
  let maximumContentSize = CGSize(
    width: max(1, visibleFrame.width - chrome.width),
    height: max(1, visibleFrame.height - chrome.height)
  )

  return CGSize(
    width: min(max(displayMinimumSize.width, visibleFrame.width * 0.76 - chrome.width), min(displayPreferredSize.width, maximumContentSize.width)),
    height: min(max(displayMinimumSize.height, visibleFrame.height * 0.72 - chrome.height), min(displayPreferredSize.height, maximumContentSize.height))
  )
}
```

Use an `NSViewRepresentable` only as a narrow bridge to access the owning `NSWindow`, set a stable identifier, and refresh display-dependent content sizes when the selected tab or screen changes. The bridge should not call frame fitting APIs:

```swift
private struct SettingsWindowAccessor: NSViewRepresentable {
  let updateTrigger: SettingsTab
  let configure: (NSWindow) -> Void

  func makeNSView(context: Context) -> NSView {
    let view = SettingsWindowAccessorView()
    view.configure = configure
    return view
  }

  func updateNSView(_ nsView: NSView, context: Context) {
    guard let view = nsView as? SettingsWindowAccessorView else { return }
    view.configure = configure

    if let window = view.window {
      configure(window)
    }
  }
}
```

Keep unit tests on pure content-size helpers. Delete tests for removed frame/origin fitting behavior when frame fitting is intentionally no longer part of the design.

## Bringing Settings Frontmost

For menu bar apps, accessory apps, LSUIElement apps, or windows opened from a status item, `openSettings()` alone may create or reveal the Settings window without making the app frontmost.

Recommended sequence when the user explicitly requested Settings:

1. Switch activation policy if the app normally runs as accessory:
   ```swift
   NSApp.setActivationPolicy(.regular)
   ```
2. Unhide and activate the app before opening Settings:
   ```swift
   NSApp.unhide(nil)
   NSRunningApplication.current.activate(options: [.activateIgnoringOtherApps])
   ```
   `NSApp.activate(ignoringOtherApps: true)` exists and may be present in older code, but it is deprecated. Prefer `NSRunningApplication.current.activate(options:)` or `NSApp.activate()` unless existing behavior specifically requires the deprecated call for a target OS.
3. Clear Settings frame autosaves if you intentionally want to override previous placement:
   ```swift
   clearSettingsWindowFrameAutosaves()
   ```
4. Call SwiftUI's `openSettings()`.
5. Run a bounded retry that finds the Settings window and orders it front.

Example bounded retry:

```swift
@MainActor
private enum SettingsWindowFocus {
  static let appSettingsWindowIdentifier = "KindowSettingsWindow"
  static let swiftUISettingsWindowIdentifier = "com_apple_SwiftUI_Settings_window"
  static let maxAttempts = 10
  static let retryDelayNanoseconds: UInt64 = 150_000_000
}

@MainActor
func showSettings(openSettings: @escaping () -> Void) {
  activateAppForSettings()
  clearSettingsWindowFrameAutosaves()
  openSettings()

  Task { @MainActor in
    await focusSettingsWindow()
  }
}

@MainActor
private func focusSettingsWindow(attempt: Int = 1) async {
  activateAppForSettings()

  if let settingsWindow = findSettingsWindow() {
    settingsWindow.deminiaturize(nil)
    settingsWindow.makeKeyAndOrderFront(nil)
    settingsWindow.orderFrontRegardless()
    activateAppForSettings()

    let isFrontmost = NSWorkspace.shared.frontmostApplication?.processIdentifier ==
      NSRunningApplication.current.processIdentifier
    if settingsWindow.isKeyWindow && isFrontmost {
      return
    }
  }

  guard attempt < SettingsWindowFocus.maxAttempts else {
    return
  }

  try? await Task.sleep(nanoseconds: SettingsWindowFocus.retryDelayNanoseconds)
  await focusSettingsWindow(attempt: attempt + 1)
}

@MainActor
private func activateAppForSettings() {
  NSApp.setActivationPolicy(.regular)
  NSApp.unhide(nil)
  NSRunningApplication.current.activate(options: [.activateIgnoringOtherApps])
}
```

`orderFrontRegardless()` can help when another app is active, but Apple documents that it does not change the key or main window. Do not rely on it alone. Pair it with app activation and `makeKeyAndOrderFront(nil)`.

## Finding the Settings Window

Use layered matching. Private SwiftUI identifiers are not API and can change, so do not depend on one private string only.

```swift
@MainActor
private func findSettingsWindow() -> NSWindow? {
  NSApp.windows.first { window in
    if window.identifier?.rawValue == SettingsWindowFocus.appSettingsWindowIdentifier {
      return true
    }

    if window.identifier?.rawValue == SettingsWindowFocus.swiftUISettingsWindowIdentifier {
      return true
    }

    guard window.isVisible else {
      return false
    }

    let title = window.title.lowercased()
    return title.contains("settings") || title.contains("preferences")
  }
}
```

Prefer an app-owned window identifier when possible. Keep title matching as a fallback only.

## Menu Bar App Caveats

- `setActivationPolicy(.regular)` may briefly show a Dock icon. This is expected if a menu bar app needs a normal Settings window to become key/frontmost.
- Switch the activation policy before `openSettings()`. Doing it afterward is more likely to produce focus races.
- If using `@Environment(\.openSettings)` inside `MenuBarExtra`, ensure the SwiftUI scene hierarchy actually provides that environment. Some apps keep a tiny hidden `Window` scene to host menu bar content; scene declaration order can matter.
- If the app returns to accessory mode after windows close, do that in the app delegate/window lifecycle, not during the focus retry.

## Verification Checklist

- Launch fresh, open Settings once, and confirm it appears without old-position flash.
- Move Settings, quit/relaunch, open Settings, and confirm the intended restoration policy still applies.
- Open Settings while another app is frontmost and confirm this app becomes frontmost.
- Open Settings twice in the same launch and confirm the existing Settings window is raised, not duplicated.
- Test on the minimum deployment target. macOS 15-only scene modifiers must not break macOS 14 builds.
- Run the relevant build/test command after changing any app code.
