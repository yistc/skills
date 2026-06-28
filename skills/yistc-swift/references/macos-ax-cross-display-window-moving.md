# macOS AX Cross-Display Window Moving

Agent rules for moving other apps' windows with Accessibility APIs.

## Scope
- Use for `AXUIElement` window move/resize, `AXPosition`, `AXSize`, cross-display mapping, `NSScreen.visibleFrame`, `CGDisplayBounds`, Dock/menu bar safe area, old-screen blank strips, and visible resize jank.
- Public AX has no reliable atomic setFrame. Treat frame writes as separate position and size operations.

## Display Geometry
- AX/CG window coordinates use top-left style CG display bounds. Do not assume `NSScreen.frame` contains AX points.
- Resolve display ownership with `CGDisplayBounds(displayID).contains(point)`, usually using the window center.
- `NSScreen.visibleFrame` can omit a secondary display menu bar. If needed, detect WindowServer `Menubar` windows from `CGWindowListCopyWindowInfo`.
- Ignore Dock for non-main display moves unless the product explicitly wants Dock-aware secondary behavior.
- Do not treat `(-1920,30,1920,1050)` on a `1920x1080` display as Dock loss. That is normally menu bar exclusion.
- Real old-Dock-strip loss looks like old-screen size carried to the new screen, e.g. `(0,33,1512,856)` -> `(-1920,30,1512,856)`.

## Frame Mapping
- Compute the target app-acceptable frame before AX writes. Prefer target visible/safe frame over raw full bounds.
- For maximized-like source windows, map directly to the target visible/safe frame.
- For ordinary windows, map proportionally between source and target mapping frames.
- Preserve main-display Dock safe area. Restore horizontal/bottom Dock space when mapping to non-main displays unless requirements say otherwise.

## AX Write Strategy
- Use deterministic `size -> position -> size` for cross-display moves. Rectangle uses the same pattern because macOS can clamp size to the current display before ownership changes.
- Temporarily disable app-level `AXEnhancedUserInterface` during frame writes when it is enabled; restore it after a short bounded delay.
- If readback width/height still differs after a real cross-display move, allow one short async final-size correction. Do not add blocking `Thread.sleep`, animation waits, or multi-round retries.
- Do not capture non-Sendable `AXUIElement` in async closures under strict concurrency. Re-resolve narrowly by pid/window identity or keep the correction synchronous.

## Window Selection
- Prefer `AXFocusedWindow`, then `AXMainWindow`, then filtered `AXWindows`.
- Filter hidden, minimized, non-`AXWindow`, `AXUnknown`, and tiny utility windows before movement.
- Do not move Dock windows. Dock is not app window state and cannot be managed through this feature.

## Logging
- Log `moving` only when a display move will be attempted.
- Log `skipping already on target display` when no move is performed.
- Include source display, target display, mapping mode, requested frame, and actual readback frame.
- Logs before display/target checks are misleading. Avoid them.

## Verification
- Verify with AX readback, not screenshots or computed frames only.
- Record source frame, source display, target mapping frame, immediate actual frame, and delayed actual frame if a correction is scheduled.
- Test at least one Dock-constrained main display -> non-main display maximized-like window.
- Confirm final size matches the target visible/safe frame, not the old display's visible frame.

## Performance
- Normal move should be fixed-cost: frame read, small number of AX writes, frame readback.
- Any correction must be bounded to one async pass.
- Avoid CGWindowList scans in tight loops. Cache per operation or per display-change cycle if movement becomes frequent.
