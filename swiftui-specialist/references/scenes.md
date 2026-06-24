# Scene & presentation lifetime & teardown

Scene, sheet, navigation, and window presentations have **categorically different** teardown semantics. There is no single "@State lifetime = view lifetime" rule — that principle is domain-wrong for `sheet(item:)`, navigation, `.id()`, and iPad scenes, which conflate distinct lifecycles. Find the mechanism you are using below; each subsection gives its applicability and its teardown recipe.

## Framing: `@State` lifetime is the structural-identity node

`@State` lifetime is tied to the view's **structural-identity node** in the view graph — the node persists across `body` re-evaluation and is released when the node leaves the graph. It is **not** the struct value (recreated freely on every `body` pass) and **not** "the view" as a colloquial whole.

Identity is determined by **view type + graph position + any `.id(value)` modifier**:

- Changing `.id(value)` **replaces the node and resets `@State`** even at a stable graph position. `.id()` targets the modified view, **not** sibling subtrees.
- `@State` **initial value is captured once**, at node creation. Re-running `body` does not re-run the initializer; assigning a new initial expression has no effect on an already-created node.
- An `if` / `switch` **branch flip** moves the view to a different structural position → `@State` resets. (To preserve state across a conditional, keep the view at one position and drive the difference with data, not by swapping branches.)
- `@StateObject`'s `deinit` fires when **its node** leaves the graph — this is distinct from `@State` value release, and is the hook for class-backed teardown.
- `ForEach` row identity follows the element `id` (see `references/foreach.md` for the stable-and-unique identity rules and the index-as-identity anti-pattern).

## Async scoping: `.task` is primary; carve-out for work that outlives the view

`.task` cancellation is the **primary** tool for view-scoped async service lifetime — SwiftUI cancels the task automatically when the view leaves the graph. Use it for anything whose lifetime should match the view.

**Carve-out — work that must *outlive* the view** (in-flight uploads, audio sessions, background location / health):

- **Preferred:** an actor or an `@Observable` in the environment that **owns the `Task`**. The longer-lived owner survives the view's teardown; the view only observes.
- **Last resort:** a detached `Task` with explicit cancellation + `Sendable` handling. (`Task.detached` is usually a smell — reach for the owned-by-environment form first.)

## sheet / fullScreenCover / popover

- **`sheet(item:)` tears down `@State` mid-session on item-*identity* change** (not only on explicit dismiss): if the bound item's identity changes while the sheet is up, SwiftUI tears down the old sheet's content (and its `@State`) and builds a fresh one for the new item. Do not assume sheet `@State` survives an item swap.
- **`sheet(isPresented:)` does *not* reset on re-presentation:** dismissing and re-presenting via the same boolean does not guarantee fresh `@State`. If you need a clean slate per presentation, drive from `sheet(item:)` with a fresh-identity item, or apply `.id()` to force a new node.
- **`fullScreenCover` uses a separate hosting root** — some environment propagation differs from the presenting view; re-inject environment values the covered content needs rather than assuming inheritance.
- **iPad `.popover` is user-dismissable** (tap-outside) and **not interceptable** — you cannot block or veto the dismissal, so don't rely on a confirmation gate before teardown.
- **`inspector()` toggles / collapses** rather than presenting modally; the inspector content's **`onDisappear` fires on collapse**.
- **`onDismiss` fires *post*-disappear** — by the time it runs the presented view is already gone. **Don't read final `@Binding` values in `onDismiss`** for interactive dismiss; capture what you need before teardown.

## Navigation destinations

- A **path-value-identity change re-creates the destination** (resets its `@State`): pushing a value with a different identity builds a fresh destination node.
- A **full-path replacement tears down all stacked destinations** — assigning a whole new path array releases every destination currently on the stack.
- **`navigationDestination(for:)` must be registered on the `NavigationStack` or a stable ancestor, never on a transient child** (a known identity bug — registering it on a child that can leave the graph causes the destination to silently stop resolving).
- **`NavigationSplitView` columns** have distinct lifetimes: the **sidebar is always present**; **content / detail are re-created on selection identity** change; the **compact size class collapses the split view to a stack**, changing which columns exist.

## macOS `WindowGroup` / `Window`

- There is **no native window-close hook** (no `windowWillClose`-equivalent SwiftUI modifier through current macOS), **but the root view's `onDisappear` *does* fire on macOS window close** — this is the simpler hook for window-scoped teardown.
- **`NSWindow.willCloseNotification` is a global broadcast** — every window's view receives it. Filtering by the `WindowGroup` scene-id prefix matches **all sibling windows** of that group, not just the one that closed, so it is not a per-window signal on its own.
- The **`Settings`-scene "won't quit after last window" trap:** an app with a `Settings` scene can keep the process alive after the last visible window closes; account for it when wiring quit/teardown behavior.
- **`isolated deinit` (SE-0371)** is an actor-isolation feature for safely running `@MainActor` teardown when the model deallocates. It is **useless against retain cycles** — a `.task` closure or a `NotificationCenter` observer that keeps the model alive defeats it (the deinit simply never runs). It is a **supplement to, not a replacement for, breaking ownership cycles**: first ensure the model is actually released (break the cycle), *then* use `isolated deinit` to run the `@MainActor` cleanup safely.

### AppKit bridge: native `NSWindow` title-bar tabs

(Home for the native `NSWindow` title-bar-tabs AppKit bridge — when a SwiftUI macOS window needs system window-tab grouping that `WindowGroup` does not expose, bridge to `NSWindow` here.)

## iPad multi-scene

- **`@State` is lost on scene serialization / termination** (not just on backgrounding) — when the system serializes and later restores a scene, in-memory `@State` is gone. **Use `@SceneStorage`** for state that must survive scene restoration.
- **`@SceneStorage` keys are app-global** — if multiple scenes store the same logical key, they collide. **Namespace the key per-scene manually** (e.g. prefix with a scene identifier) when more than one scene persists the same logical value.

## Persistence lifetime

For `modelContainer` / `@Query` lifetime (where the SwiftData store is attached, and how query results track the container), see the `swiftdata-pro` skill.
