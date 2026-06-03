# Data flow, shared state, and property wrappers

It is important that SwiftUI body code and logic code be kept separate in order to make code easier to read, write, and maintain. That usually means placing code into methods rather than inline in the `body` property, but often also means carving functionality out into separate `@Observable` classes.

These rules help ensure code is efficient and works well in the long term.


## Shared state

- `@Observable` classes must be marked `@MainActor` unless the project has Main Actor default actor isolation. Flag any `@Observable` class missing this annotation.
- All shared data should use `@Observable` classes with `@State` (for ownership) and `@Bindable` / `@Environment` (for passing).
- Strongly prefer not to use `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, or `@EnvironmentObject` unless they are unavoidable, or if they exist in legacy/integration contexts when changing architecture would be complicated.


## Local state

- `@State` should be marked `private` and only owned by the view that created it.
- If a view stores a class instance that contains expensive-to-recompute data, e.g. `CIContext`, it can be stored using `@State` even though it is not an observable object. This effectively uses `@State` as a cache – storing something persistently, but not doing any change tracking on it since it's not an observable object.


## Bindings

- When a child view both reads and writes state owned by its parent, pass a `@Binding` (e.g. `@Binding var isPresented`), not an `onChange`/callback closure. Callback closures are for one-shot actions with no parent state to mutate (mirroring `Button`'s `action:`).
- Prefer `$`-prefixed projected bindings over manually-constructed `Binding(get:set:)` closures inline in a view body — route side effects through `onChange()` and type projections through a `Binding` extension (or `*(item:)` APIs) instead.
- If the user needs to enter a number into a `TextField`, bind the `TextField` to a numeric value such as `Int` or `Double`, then use its `format` initializer like this: `TextField("Enter your score", value: $score, format: .number)`. Apply either `.keyboardType(.numberPad)` (for integers) or `.keyboardType(.decimalPad)` (for floating-point numbers) as appropriate. Using the modifier alone is *not* sufficient.


## Working with data

- Prefer to make structs conform to `Identifiable` rather than using `id: \.someProperty` in SwiftUI code.
- Never attempt to use `@AppStorage` inside an `@Observable` class, even if marked `@ObservationIgnored` – it will *not* trigger view updates when a change happens.


## macOS app lifecycle

- A SwiftUI macOS app that uses `Settings { ... }` and wants Terminal.app/iTerm-style "quit when the last main window closes" cannot return unconditional `true` from `applicationShouldTerminateAfterLastWindowClosed(_:)`. The `Settings` scene creates a real `NSWindow` that counts toward "open windows," so closing the main window with Settings still up keeps the app alive. Filter `NSApp.windows` by identifier (e.g., `window.identifier?.rawValue.contains("Settings") != true`) and return `nonSettings.isEmpty`. If quit-on-last-window is not desired (the macOS default), omit the `NSApplicationDelegateAdaptor` entirely.
- Prefer `@Environment(\.dismissWindow)` (macOS 14+) over imperative `NSApp.keyWindow?.close()` for closing the host window from a SwiftUI view — the no-arg form dismisses **this** action's host window, no `keyWindow` lookup race.

## SwiftData

- If you only need the number of items matching a query, consider `ModelContext.fetchCount()` with a fetch descriptor. This will *not* live update if the data changes unless something else triggers the update, such as `@Query`, so it should be used carefully.

For more help with SwiftData, suggest the [SwiftData Pro agent skill](https://github.com/twostraws/swiftdata-agent-skill).

## If the project uses SwiftData with CloudKit

- Never use `@Attribute(.unique)`.
- Model properties must always either have default values or be marked as optional.
- All relationships must be marked optional.
