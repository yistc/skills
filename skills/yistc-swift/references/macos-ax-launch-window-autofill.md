# macOS AX Launch Window Auto-Move

Agent rules for applying auto move/maximize behavior to a newly launched app's first window.

## Scope
- Use when an app should be moved or maximized on opening, especially after the app was fully quit.
- Relevant APIs include `NSWorkspace.didLaunchApplicationNotification`, `AXObserver`, `AXObserverAddNotification`, `kAXWindowCreatedNotification`, `kAXMainWindowChangedNotification`, and `kAXWindowResizedNotification`.
- This is different from the case where the app is already running with no windows: in that case the AX observer may already exist, so `kAXWindowCreatedNotification` is easier to catch.

## Root Cause Pattern
- `didLaunchApplicationNotification` can arrive before the target app has created its first user window.
- The first window can also be created before AX notifications finish registering.
- Synchronous AX notification registration can be slow for some apps and can block the app's main actor if done directly in the launch monitor.
- The user-visible symptom is: creating a new window for an already-running app works, but launching a fully quit app does not auto move/maximize its first window.

## Preferred Design
- On launch, first mark the app as pending if it has auto move/maximize behavior to apply.
- Immediately perform one window check in case the first window already exists.
- Create the AX observer, then register AX notifications in a background task.
- After registration completes, hop back to the main actor and perform exactly one catch-up window check.
- Keep AX callbacks for later readiness events: window created, main window changed, and window resized.
- Remove the pending pid once movement is applied, the app terminates, or the feature/license gate is closed.

## Avoid
- Do not synchronously call `AXObserverAddNotification` from `didLaunchApplicationNotification` handling on the main actor.
- Do not use short polling as the first solution. It can be a last resort only after proving AX launch/register/catch-up events are insufficient.
- Do not rely only on `kAXWindowCreatedNotification`; the first window may be created before registration completes.
- Do not move all windows unconditionally on launch. First check product gates, app configuration, and whether the app has suitable windows.

## Implementation Shape
```swift
private var pendingLaunchAutoMovePIDs: Set<pid_t> = []

private func prepareLaunchAutoMove(for app: ActiveApp) {
  guard shouldAutoMoveAppWindows(for: app) else { return }
  pendingLaunchAutoMovePIDs.insert(app.pid)
  applyPendingLaunchAutoMoveIfPossible(for: app, reason: "didLaunch")
}

private func applyPendingLaunchAutoMoveIfPossible(for app: ActiveApp, reason: String) {
  guard pendingLaunchAutoMovePIDs.contains(app.pid) else { return }

  guard featureGateAllowsAutoMove() else {
    pendingLaunchAutoMovePIDs.remove(app.pid)
    return
  }

  guard AXUtils.getAppWindows(pid: app.pid)?.isEmpty == false else { return }

  moveAppWindowsToSpecifiedDisplay(pid: app.pid)
  pendingLaunchAutoMovePIDs.remove(app.pid)
}
```

```swift
prepareLaunchAutoMove(for: app)
setupAXObserver(for: app, applyPendingLaunchAutoMoveAfterRegistration: true)
```

```swift
Task.detached { [weak self] in
  guard let self else { return }
  self.registerAXNotifications(observer, for: app)

  if applyPendingLaunchAutoMoveAfterRegistration {
    await MainActor.run {
      self.applyPendingLaunchAutoMoveIfPossible(for: app, reason: "observerRegistered")
    }
  }
}
```

## Concurrency Notes
- Keep pending state and window movement on the main actor when they affect UI-owned app state.
- Treat `AXObserver`, `AXUIElement`, and raw pointers as non-Sendable. Keep captures narrow and do not let background tasks mutate main-actor state directly.
- `Task.detached` is acceptable here only because AX notification registration is synchronous external-system work that can block the caller; return to `MainActor.run` before touching actor-isolated state.
- If strict concurrency diagnostics appear, first preserve the same architecture: background registration, main-actor catch-up. Do not regress to synchronous registration just to silence warnings.

## Verification
- Test both cases:
  - The target app is fully quit, then launched.
  - The target app is already running with all windows closed, then a new window is opened.
- Test with a known app such as Apple Notes and confirm the first window auto move/maximize behavior.
- Test multiple apps launching close together and confirm the app running this logic does not freeze.
- Confirm the behavior still respects license/feature gates and app-specific configuration.
