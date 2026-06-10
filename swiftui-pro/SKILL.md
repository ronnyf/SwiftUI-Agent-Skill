---
name: swiftui-pro
description: Comprehensively reviews SwiftUI code for best practices on modern APIs, maintainability, and performance. Use when reading, writing, or reviewing SwiftUI projects.
license: MIT
metadata:
  author: Paul Hudson
  version: "1.3"
---

Review Swift and SwiftUI code for correctness, modern API usage, and adherence to project conventions. Report only genuine problems — do not nitpick or invent issues.

## Review process

1. Check deprecated and soft-deprecated API — §API below, then invoke `agentic:swiftui-specialist` and load `references/soft-deprecation.md` + `references/soft-deprecated-apis.md`.
1. Check view structure, modifiers, and animations — §Views below, then invoke `agentic:swiftui-specialist` and load `references/structure.md`, `references/modifiers.md`, `references/animations.md`.
1. Validate data flow — §Data Flow below, then invoke `agentic:swiftui-specialist` and load `references/dataflow.md` + `references/foreach.md` for deep `@Observable`, `@Binding`, and collection-identity rules.
1. Ensure navigation is updated and performant — §Navigation.
1. Ensure the code meets Apple's Human Interface Guidelines — §Design.
1. Validate accessibility compliance — §Accessibility.
1. Ensure the code runs efficiently — §Performance.
1. Check environment values and `@Entry` usage — invoke `agentic:swiftui-specialist` and load `references/environment.md`.
1. Validate localization — invoke `agentic:swiftui-specialist` and load `references/localization.md`.
1. Quick validation of Swift code — §Swift.
1. Final code hygiene check — §Hygiene.
1. **If the deployment target is iOS/macOS/watchOS/tvOS/visionOS 27 or later:** invoke `agentic:swiftui-whats-new-27` and load the relevant references for `@State` macro migration, `@ContentBuilder`, and SDK 27 hard-deprecations.

If doing a partial review, load only the relevant sections and skills.


## Core Instructions

- iOS 26 exists, and is the default deployment target for new apps.
- Target Swift 6.2 or later, using modern Swift concurrency.
- As a SwiftUI developer, the user will want to avoid UIKit unless requested.
- Do not introduce third-party frameworks without asking first.
- Break different types up into different Swift files rather than placing multiple structs, classes, or enums into a single file.
- Use a consistent project structure, with folder layout determined by app features.


## §API

- Always use `foregroundStyle()` instead of `foregroundColor()`.
- Always use `clipShape(.rect(cornerRadius:))` instead of `cornerRadius()`.
- Always use the `Tab` API instead of `tabItem()`.
- Never use the `onChange()` modifier in its 1-parameter variant; use the 2-parameter or 0-parameter variant.
- Do not use `GeometryReader` if a newer alternative works: `containerRelativeFrame()`, `visualEffect()`, or the `Layout` protocol.
- When designing haptic effects, prefer `sensoryFeedback()` over older UIKit APIs such as `UIImpactFeedbackGenerator`.
- Use the `@Entry` macro to define custom `EnvironmentValues`, `FocusValues`, `Transaction`, and `ContainerValues` keys.
- Strongly prefer `overlay(alignment:content:)` over the deprecated `overlay(_:alignment:)`.
- Never use `.navigationBarLeading` / `.navigationBarTrailing`; use `.topBarLeading` / `.topBarTrailing`.
- Prefer automatic grammar agreement for English, French, German, Portuguese, Spanish, and Italian: `Text("^[\(n) person](inflect: true)")`.
- You can fill and stroke a shape with two chained modifiers — no overlay needed (iOS 17+).
- When referencing images from an asset catalog, prefer the generated symbol asset API: `Image(.avatar)` not `Image("avatar")`.
- When targeting iOS 26+, use the native `WebView` (requires `import WebKit`) instead of hand-wrapped `WKWebView`.
- `ForEach` over an `enumerated()` sequence: use `ForEach(items.enumerated(), id: \.element.id)` directly, do not convert to an array first.
- Use `.scrollIndicators(.hidden)` not `showsIndicators: false`.
- Never use `Text` concatenation with `+`; use text interpolation instead.
- If `ObservableObject` is required (e.g. Combine debouncer), ensure `import Combine` is present — SwiftUI no longer re-exports it.

*For soft-deprecated patterns, invoke `agentic:swiftui-specialist` and load `references/soft-deprecation.md` + `references/soft-deprecated-apis.md`.*


## §Views

- Strongly prefer separate `View` structs over computed properties or `@ViewBuilder` methods that return `some View`. Computed properties share the parent's invalidation boundary; dedicated structs own their own `@State`, lifecycle, and `#Preview`.
- Flag excessively long `body` properties — break into extracted subviews.
- Button actions should be extracted into separate methods, not inlined in view bodies.
- Business logic should not live inline in `task()`, `onAppear()`, or elsewhere in `body`.
- Drive async work tied to a view's lifetime from `.task` / `.task(id:)` — SwiftUI cancels automatically when the view leaves the view graph. Do not store a `Task` as a property and cancel by hand.
- Each type (struct, class, enum) should be in its own Swift file.
- Prefer `TextField` with `axis: .vertical` over `TextEditor` unless a full-screen editing experience is required.
- Use `#Preview` — not the legacy `PreviewProvider` protocol.
- When using `TabView(selection:)`, bind to an enum property, not an integer or string.
- A `.task { }` on a conditional branch is cancelled when `@Observable` state changes swap branches. Lifecycle tasks must be attached to the always-present outer container; use `.task(id:)` with `@State` keying re-firing.
- `Tab(_:systemImage:value:content:)` requires a non-optional `selection` binding (iOS 18+ / macOS 15+).
- Never use `animation(_ animation: Animation?)`; always provide a value: `.animation(.bouncy, value: score)`.
- Chain animations with `withAnimation { } completion: { withAnimation { } }`, not multiple `withAnimation` calls with delays.
- Prefer `@Animatable` macro over manual `animatableData`.
- When rendering to images, prefer `ImageRenderer` over `UIGraphicsImageRenderer`.

*For view factoring, invalidation boundaries, init costs, and single-child `Group` anti-pattern, invoke `agentic:swiftui-specialist` and load `references/structure.md`.*
*For `ForEach` / `List` / `Table` identity and collection performance, load `references/foreach.md`.*


## §Data Flow

- Keep SwiftUI body code and logic separate — extract into `@Observable` classes.
- `@Observable` classes must be marked `@MainActor` unless the project has Main Actor default actor isolation.
- Prefer `@Observable` + `@State` (ownership) + `@Bindable` / `@Environment` (passing). Avoid `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject` unless unavoidable.
- "Stale value / didn't update" bugs are almost always state-consistency problems — an `@State` mirror drifted from source of truth. Fix by removing the desyncable mirror, not by swapping `.task(id:)` and `.onChange`.
- `@State` should be `private` and owned by the view that created it.
- When a child both reads and writes parent state, pass `@Binding` — not an `onChange`/callback closure. Closures are for one-shot actions with no parent state to mutate.
- Prefer `$`-prefixed projected bindings over `Binding(get:set:)` closures inline in a view body.
- For numeric `TextField`, bind to `Int`/`Double` with the `format:` initializer and `.keyboardType(.numberPad)` / `.keyboardType(.decimalPad)`.
- Prefer `Identifiable` conformance over `id: \.someProperty` in SwiftUI code.
- Never use `@AppStorage` inside an `@Observable` class — it will not trigger view updates.
- macOS: use `@Environment(\.dismissWindow)` (macOS 14+) over `NSApp.keyWindow?.close()`.

*For deep `@Observable` per-property tracking, collection granularity, `@Binding` KeyPath patterns, and `onChange` isolation, invoke `agentic:swiftui-specialist` and load `references/dataflow.md`.*


## §Navigation

- Use `NavigationStack` or `NavigationSplitView`; flag all use of deprecated `NavigationView`.
- Prefer `navigationDestination(for:)` over `NavigationLink(destination:)`; flag the old pattern.
- Never mix `navigationDestination(for:)` and `NavigationLink(destination:)` in the same hierarchy.
- `navigationDestination(for:)` must be registered once per data type.
- Always attach `confirmationDialog()` to the UI that triggers it.
- If an alert has only a single dismiss "OK" button with no action, the button can be omitted entirely.
- Prefer `sheet(item:)` over `sheet(isPresented:)` when presenting optional data.
- When `sheet(item:)` accepts the item as its only init parameter, prefer `sheet(item: $item, content: SomeView.init)`.


## §Design

- Prefer to place standard fonts, sizes, colors, spacing, padding, rounding, and timing into a shared enum of constants for uniformity.
- Never use `UIScreen.main.bounds`; prefer `containerRelativeFrame()`, `visualEffect()`, or (as last resort) `GeometryReader`.
- Avoid fixed frames unless content fits neatly — they break across device sizes and Dynamic Type.
- Apple's minimum tap area on iOS is 44×44; enforce strictly.
- Prefer `ContentUnavailableView` when data is missing or empty.
- Use `ContentUnavailableView.search` (not `.search(text:)`) with `searchable()` for empty results.
- Prefer `Label` over `HStack` for icon + text side by side.
- Prefer system hierarchical styles (secondary/tertiary) over manual opacity.
- In `Form`, wrap `Slider` in `LabeledContent` for correct layout.
- When using `RoundedRectangle`, `.continuous` is the default — no need to specify it.
- Use `bold()` over `fontWeight(.bold)`; only use `fontWeight()` for weights other than bold when there is a specific reason.
- Avoid hard-coded padding and stack spacing unless requested.
- Avoid `UIColor` in SwiftUI code; use SwiftUI `Color` or asset catalog colors.
- `.caption2` is extremely small — use sparingly. `.caption` is also small; use carefully.


## §Accessibility

- Respect user accessibility settings for fonts, colors, animations.
- Do not force specific font sizes; prefer Dynamic Type.
- If a custom font size is needed: use `@ScaledMetric` for iOS ≤ 18; `.font(.body.scaled(by:))` is also available on iOS 26+.
- Flag images with unclear VoiceOver readings. Decorative: `Image(decorative:)` or `.accessibilityHidden(true)`. Informative: add `.accessibilityLabel()`.
- When "Reduce Motion" is on, replace motion-based animations with opacity.
- For frequently changing button labels, recommend `accessibilityInputLabels()`.
- Buttons with image labels must always include text, even if visually hidden.
- When color is an important differentiator, respect `.accessibilityDifferentiateWithoutColor`.
- Same for `Menu`: prefer `Menu("Options", systemImage: "ellipsis.circle") { }` over image-only.
- Never use `onTapGesture()` unless tap location or count is needed. Use `Button` instead.
- If `onTapGesture()` must be used, add `.accessibilityAddTraits(.isButton)`.


## §Performance

- When toggling modifier values, prefer ternary expressions over `if/else` branching to preserve structural identity.
- Avoid `AnyView` unless absolutely required; use `@ViewBuilder`, `Group`, or generics.
- If a `ScrollView` has an opaque static background, use `scrollContentBackground(.visible)`.
- Break views into dedicated `View` structs — computed properties don't create new invalidation boundaries.
- Keep view initializers minimal; move non-trivial work into `.task()`.
- Assume `body` is called frequently — move sorting/filtering out of `body`.
- Avoid storing `DateFormatter` etc. as properties; use `Text(date, format: .dateTime…)` instead.
- Avoid expensive inline transforms in `List`/`ForEach` initializers.
- Prefer deriving transformed data with `let`, or caching in `@State` with explicit invalidation logic.
- For large data sets in `ScrollView`, use `LazyVStack`/`LazyHStack`.
- Prefer `.task()` over `onAppear()` for async work — cancelled automatically on disappear.
- Avoid storing escaping `@ViewBuilder` closures on views; store the built view result instead.


## §Swift

- Prefer Swift-native string methods: `replacing("a", with: "b")` not `replacingOccurrences(of:with:)`.
- Prefer modern Foundation API: `URL.documentsDirectory`, `appending(path:)`.
- Never use C-style number formatting (`String(format: "%.2f", value)`); use `FormatStyle` APIs.
- Prefer static member lookup: `.circle` over `Circle()`, `.borderedProminent` over `BorderedProminentButtonStyle()`.
- Avoid force unwraps and force `try` unless truly unrecoverable; prefer `fatalError()` with a description.
- Text filtering on user input: use `localizedStandardContains()`, not `contains()` or `localizedCaseInsensitiveContains()`.
- Prefer `Double` over `CGFloat` except with optionals or `inout`.
- Count matching elements with `count(where:)`, not `filter { }.count`.
- Prefer `Date.now` over `Date()`.
- `import SwiftUI` already imports `UIKit`/`AppKit` — no extra import needed for `UIImage`/`NSImage`.
- For person names, prefer `PersonNameComponents` with modern formatting.
- Centralize repeated sort closures by conforming to `Comparable`.
- Use `"y"` not `"yyyy"` for year format strings when formatting for display.
- Parse dates with `Date(myString, strategy: .iso8601)`.
- Flag errors triggered by user actions that are swallowed silently.
- Use `if let value {` shorthand over `if let value = value {`.
- Omit `return` for single-expression functions; use `if`/`switch` as expressions.
- Swift Concurrency: always prefer `async`/`await` over closure-based variants.
- Never use `DispatchQueue`; use Swift Concurrency.
- Never use `Task.sleep(nanoseconds:)`; use `Task.sleep(for:)`.
- Flag mutable shared state not protected by actor or `@MainActor`.
- Apply strict concurrency rules; flag `@Sendable` violations and data races.
- `Task.detached()` is often a bad idea — check any usage carefully.


## §Hygiene

- Never include secrets (API keys etc.) in the repository.
- Comments and documentation comments should be present where logic is not self-evident.
- Unit tests for core logic; UI tests only where unit tests are not possible.
- Never use `@AppStorage` for usernames, passwords, or sensitive data — use Keychain.
- If SwiftLint is configured, it should return no warnings or errors.
- If the project uses `Localizable.xcstrings`, prefer symbol keys with `extractionState: manual`.
- If the Xcode MCP is configured, prefer its tools (`RenderPreview`, `DocumentationSearch`) over generic alternatives.


## Output Format

Organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated.
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

### Summary format

1. **Severity (Critical/Important/Suggestion):** description — file:line
