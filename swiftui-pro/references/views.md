# SwiftUI Views

- Strongly prefer to avoid breaking up view bodies using computed properties or methods that return `some View`, even if `@ViewBuilder` is used. Extract them into separate `View` structs instead, placing each into its own file.
- Why dedicated structs beat `@ViewBuilder` helpers (funcs or computed `some View` props): (1) **previewable in isolation** — a `View` struct can have its own `#Preview`; a helper cannot, so you would have to preview the whole parent; (2) **cheap** — `View`s are lightweight value types the framework creates and diffs in bulk, so many small structs is the intended grain, not a smell; (3) **composable and state-owning** — a struct is an addressable, parameterized, reusable unit that can own its own `@State` / `@FocusState` / lifecycle (`.task`, `.onAppear`), whereas a helper owns no state (it pushes that state up into its parent, coupling unrelated concerns), is callable only within its parent, and is a deferred extraction — technical debt the day you need to reuse it. (Performance is a further reason — see `performance.md`.)
- Flag `body` properties that are excessively long; they should be broken into extracted subviews.
- If the user has created a handful of small, private helper `some View` properties for structural readability, and they both belong to the same concern as `body` and would fit in `body` at an acceptable length if inlined, these can be left alone. Otherwise, they should be extracted to new `View` structs.
- Button actions should be extracted from view bodies into separate methods, to avoid mixing layout and logic.
- Similarly, general business logic should not live inline in `task()`, `onAppear()` or elsewhere in `body`.
- Prefer to place view logic into view models or similar, so it can be tested. For more help with testing, suggest the [Swift Testing Pro agent skill](https://github.com/twostraws/swift-testing-agent-skill).
- Each type (struct, class, enum) should be in its own Swift file. Flag files containing multiple type definitions.
- Unless a full-screen editing experience is required, prefer using `TextField` with `axis: .vertical` to using `TextEditor`, because it allows placeholder text. If a specific minimum height is required for `TextField`, use something like `lineLimit(5...)`.
- If a button action can be provided directly as an `action` parameter, do so. For example: `Button("Label", systemImage: "plus", action: myAction)` is preferred over `Button("Label", systemImage: "plus") { action() }`.
- When rendering SwiftUI views to images, strongly prefer `ImageRenderer` over `UIGraphicsImageRenderer`.
- `#Preview` should be used for previews, not the legacy `PreviewProvider` protocol.
- When using `TabView(selection:)`, use a binding to a property that stores an enum rather than an integer or string. For example, `Tab("Home", systemImage: "house", value: .home)` is better than `Tab("Home", systemImage: "house", value: 0)`.
- Strongly prefer to avoid breaking up view bodies using computed properties or methods that return `some View`, even if `@ViewBuilder` is used. Extract them into separate `View` structs instead, placing each into its own file. (Yes, this is repeated, but it’s so important it needs to be mentioned twice.)
- A `.task { ... }` modifier attached to a view that lives inside a conditional `if/else` branch is cancelled when an `@Observable` state change causes SwiftUI to switch branches — the replacement view doesn't inherit the previous branch's task. Lifecycle-driving tasks (bootstrap, long-running drains, retry loops) must be attached to the always-present outer container (`Group`, `NavigationStack`, etc.), not to a conditional inner view. Use `.task(id: trigger)` on the outer container with `@State` keying re-firing.
- The `Tab(_:systemImage:value:content:)` view-builder API (macOS 15+ / iOS 18+) requires a NON-Optional `selection` binding — `@State var selection: SelectionValue?` won't type-check against the modern Tab-builder closure. Seed selection synchronously (e.g., to the first item's id in `init`) and never let it go nil. The legacy `.tag(...)` + `tabItem(...)` shape accepted `Optional`, the modern shape does not.
- `tabViewStyle(.tabBarOnly)` is macOS 15+ — produces a clean tab strip below the title bar. `.sidebarAdaptable` and `.grouped` are also macOS 15+. On older deployment targets only `.automatic` and `.page` are available; flag accordingly.
- For native `NSWindow` title-bar tabs (Safari / Terminal.app / Xcode-style chrome with drag-reorder + tear-off + Window-menu integration), there is **no public SwiftUI modifier**. SwiftUI sets `tabbingIdentifier` / `tabbingMode` privately on each `WindowGroup` window. To force tabs (independent of the user's "Prefer tabs" setting), bridge to AppKit via an `NSViewRepresentable` window-accessor and set `NSWindow.tabbingMode = .preferred`. If you only need an in-content tab strip (without title-bar chrome), prefer `TabView` and skip the AppKit hop.


## Animating views

- Strongly prefer to use the `@Animatable` macro over creating `animatableData` manually – the macro automatically adds conformance to the `Animatable` protocol and creates the correct `animatableData` property. If some properties should not or cannot be animated (e.g. Booleans, integers, etc), mark them `@AnimatableIgnored`.
- Never use `animation(_ animation: Animation?)`; always provide a value to watch, such as `.animation(.bouncy, value: score)`.
- Chaining animations must be done using a `completion` closure passed to `withAnimation()`, rather than trying to execute multiple `withAnimation()` calls using delays.

For example:

```swift
Button("Animate Me") {
    withAnimation {
        scale = 2
    } completion: {
        withAnimation {
            scale = 1
        }
    }
}
```
