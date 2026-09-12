---
name: swiftui-specialist
description: "Comprehensive SwiftUI umbrella skill — idiomatic patterns, modern API, data flow, view structure, performance, accessibility, localization, and SDK 27 migration. Use when writing, reviewing, or editing any SwiftUI code, or answering SwiftUI questions."
license: MIT
metadata:
  authors: "Apple; Paul Hudson"
  version: "2.4.5"
---

Apple authored the idiomatic-pattern guidance here and in `references/dataflow.md`, `structure.md`, `foreach.md`, `environment.md`, `localization.md`, `animations.md`, `modifiers.md`, and the soft-deprecation references. It is more current than general SwiftUI knowledge, so where the two disagree, this wins. This skill covers the whole SwiftUI surface: the inline §-sections are the essentials, `references/` carries the depth — read the reference for whatever the code actually touches rather than reaching for a second SwiftUI skill. Several idioms here (`@Binding` over callback closures, separate `View` types over computed properties, per-property `@Observable` tracking) do not fall out of general knowledge.

**Two modes, and they produce different artifacts.** *Authoring* — you edit the files and report what you changed; the §-sections are design input, not a report template. *Reviewing* — you report findings and change nothing (see §Output). If the request is ambiguous ("look at this view"), treat it as review and say so in one line.

**Scoping a large codebase.** A whole-codebase sweep is worth agreeing with the user first: name the areas you would cover and what each is likely to turn up, then work the agreed list to completion in one pass rather than re-scoping mid-run. A bounded diff or a named file set needs no negotiation — read the sections and references the code touches, and go.

## What a SwiftUI review or change covers

The full specification, up front. Cover what the code touches; skip what it doesn't. Order is yours.

| Concern | Inline | Depth |
|---|---|---|
| Container/layout shape — did a shipped primitive get bypassed | §Primitive-First | the shape table in that section |
| Availability floor of every primitive you reach for, and its gate | §Primitive-First | `references/availability-gating.md` |
| View identity — what a rebuild silently discards | §Structural Identity | `references/scenes.md`, `references/modifiers.md`, `references/performance.md` |
| Deprecated and soft-deprecated API | §API | `references/soft-deprecation.md`, `references/soft-deprecated-apis.md` |
| View structure, modifiers, animations | §Views | `references/structure.md`, `references/modifiers.md`, `references/animations.md` |
| Data flow, observation, collection identity | §Data Flow | `references/dataflow.md`, `references/foreach.md` |
| Navigation; scene / presentation / window state lifetime | §Navigation, §Scenes & Windows | `references/scenes.md` |
| HIG conformance, accessibility, performance | §Design, §Accessibility, §Performance | `references/accessibility.md`, `references/performance.md` |
| Environment values, `@Entry`; localization | — | `references/environment.md`, `references/localization.md` |
| API design of a reusable component you are writing | — | `references/progressive-disclosure.md`, `references/custom-containers.md` |
| Swift idiom and hygiene | §Swift, §Hygiene | `references/swift.md` |
| Deployment target 27+ only | — | `references/state-macro.md`, `references/content-builder.md`, `references/deprecations.md` |

The first three rows are the ones general knowledge gets wrong most often, and they are cheapest to settle before the code is written rather than after.

## Core Instructions

- **"It compiles and looks right" is not the bar** — a hand-composed shape and an ungated old API both pass that bar and both forfeit what the framework gives for free. Aim at the primitive Apple built for the shape, on the newest OS available, gated.
- iOS 26 exists; default deployment target for new apps. Swift 6.2+, modern Swift concurrency.
- Avoid UIKit unless requested. No third-party frameworks without asking first. One type (struct/class/enum) per Swift file; folders by app feature.

## §Primitive-First — reach for the shipped primitive before composing one

The dominant failure mode in SwiftUI code is **hand-composing a shape SwiftUI already ships a primitive for.** `VStack { bar; Divider(); content }` compiles, renders, and passes review — and is still the wrong answer when the shape is "bar pinned to an edge," because `safeAreaBar` *is* that shape. What the hand-rolled version forfeits is not style points: platform integration, accessibility, styling hooks, scroll-edge effects, and back-deployment all come free with the primitive and have to be rebuilt (or silently go missing) without it. So the question worth answering before writing a `VStack` / `HStack` / `ZStack` / `overlay` / `background` / `GeometryReader` / custom `ViewModifier` is *what shape is this, and did Apple ship it?*

The search technique matters more than the diligence: name the shape in plain words — "bar pinned to top edge", "title-value row", "empty state", "single-choice picker", "detail pane beside content" — and run `DocumentationSearch` **on that phrase**, not on your intended implementation. Semantic matching finds a primitive from a description of what the thing does; "VStack divider bar" finds nothing, "custom bar edge" finds `safeAreaBar`. A search that genuinely comes back empty is a fine reason to compose by hand — note in the diff which phrase you searched, so the next reader doesn't redo it. A generic container is the fallback, not the default.

### Shape → primitive (non-exhaustive; search before assuming absence)

| Shape you're building | Primitive — not a stack |
|---|---|
| Bar pinned to an edge of content | `safeAreaBar(edge:)` (26+, also extends scroll-edge effects); older: `safeAreaInset(edge:)` |
| Non-bar content inset into safe area | `safeAreaInset(edge:)`, `safeAreaPadding(_:)` |
| Window/nav chrome controls | `.toolbar { ToolbarItem }`; container's own bar: `contentToolbar(for:)`; `ToolbarSpacer`, `.contentMarginsRemoved()` |
| Title + value on one line | `LabeledContent` (+ `LabeledContentStyle`) |
| Icon + text | `Label` (+ `.labelStyle`) |
| Empty / no-results / error state | `ContentUnavailableView`, `.search` |
| Single choice from a set | `Picker` (+ `.pickerStyle`) — not N Buttons |
| Grouped related buttons | `ControlGroup` |
| Expand/collapse section | `DisclosureGroup`, `Section(isExpanded:)` |
| Multi-column data | `Table` — not `ForEach` of `HStack` |
| Side pane, dismissible | `.inspector(isPresented:)`, `NavigationSplitView` |
| Search field | `.searchable(text:)` (+ `.searchScopes`, `.searchSuggestions`) |
| Share / photo / file / contact picking | `ShareLink`, `PhotosPicker`, `.fileImporter`/`.fileExporter`, `ContactAccessButton` |
| Progress / value in range | `ProgressView`, `Gauge` |
| Custom geometry across siblings | `Layout` protocol; `containerRelativeFrame`, `visualEffect` — `GeometryReader` last |
| Reading own size | `onGeometryChange(for:of:action:)` — never measure into `@State` and feed back as a frame |
| Liquid Glass grouping/morphing | `GlassEffectContainer`, `.glassEffect(_:in:)`, `.glassEffectID(_:in:)` |
| Scroll position / paging / targets | `.scrollPosition(id:)`, `.scrollTargetBehavior(.viewAligned)`, `.scrollTargetLayout()` |
| Drag-to-reorder children | `reorderable()` + `reorderContainer(for:)` (27+) — `onMove(perform:)` compiles on any `DynamicViewContent` but installs nothing outside a `List` |

**Version floor.** Adopting the newest primitive and gating it gives users on the new OS the real thing while the deployment floor holds: `if #available(anyAppleOS 26, *)` / `@available` (`anyAppleOS` is real, and collapses the per-platform matrix), older API in the `else`. Shipping only the old path because it "also works" leaves the same capability on the table as the hand-rolled stack did. Read the floor off `LSP goToDefinition` — `hover` strips `@available`, and a floor recalled from memory is wrong often enough to break a build on the deployment target. No LSP in this session, or no existing callsite to resolve from, means the DocC JSON source instead (both are listed in that reference's availability-source table); a text search or a sibling symbol's floor will give you a confident wrong answer. Neither reachable ⇒ write the gate you believe is right and label the floor `unverified:` in the diff — a guessed `#available` compiles either way, so an unlabelled guess stays silent until it reaches the older OS. Mechanics and the gating shapes: `references/availability-gating.md`.

## §Structural Identity — know what a rebuild throws away

`@State`, scroll offset, `.task` lifetime, focus and in-flight animations all live on a view's **structural-identity node** — view type + graph position + any `.id(value)`. Replacing that node discards all of it at once. "It reloaded for no reason" / "it scrolled back" / "the state reset" is what a discarded node looks like from outside, which is why this is a correctness concern rather than a performance nit.

So for any edit to a `body`, container, or bar, the useful thing to have in your head is: **which nodes in this diff can change identity, and what does each of them own?** Identity-changing sites are every `.id(value)`, every `if`/`switch` over state (including one inside a `ViewModifier` or a `@ViewBuilder` helper), every container that rebuilds its content, and every `ForEach` id expression. `#available` branches are exempt — the OS version is constant for the process, so they cannot flip. What a node owns is its `@State`, `ScrollPosition`, `.task` / `.task(id:)`, `@FocusState`, transitions — plus everything its children own.

Re-rooting is not confined to the view you modified: a container relationship (a bar hosting content over the view that flipped) can re-root a *sibling* subtree, so state is lost in a view the diff never touched. Look there before concluding the diff is clean.

**Which link to fix.** The chain is two links: *flip → sibling node re-created* (inherent, not fixable) and *re-created node → state not restored* (a property of the consuming API — usually the actual defect). The second is the one that yields. MEASURED: `.scrollPosition(id:)` requests only on a value CHANGE, so a fresh node never re-scrolls; the cure was on the scroll view — local `@State` + `.task(id:)` + one-way `scrollTo` — and it holds *with* the branch left in place. Making the flipping subtree branch-free (`opacity` ternary plus `.allowsHitTesting` and `.accessibilityHidden`; opacity gates neither, and `hidden()` has no `Bool` overload so gating it reintroduces the branch) attacks the *unfixable* link — reserve it for consumers that genuinely cannot re-assert, such as a reorderable row whose drag lift a `_ConditionalContent` does defeat. A probe that held a broken consumer constant will indict the branch every time, so re-test against the fixed consumer before writing any ban on branching down as an invariant.

Depth: `references/scenes.md` (lifetime, re-rooting, the measured scroll consequences), `references/modifiers.md` (the `.if` anti-pattern), `references/performance.md` (`_ConditionalContent`).

## §API

- **Deprecated spellings and their current replacements:** `foregroundStyle()` not `foregroundColor()` · `clipShape(.rect(cornerRadius:))` not `cornerRadius()` · `Tab` API not `tabItem()` · `.topBarLeading`/`.topBarTrailing` not `.navigationBarLeading`/`.navigationBarTrailing` · `.scrollIndicators(.hidden)` not `showsIndicators: false`.
- **Prefer:** `overlay(alignment:content:)` strongly over the deprecated `overlay(_:alignment:)` · haptics — `sensoryFeedback()` over older UIKit feedback generators (`UIImpactFeedbackGenerator` et al.) · asset-catalog images — the generated symbol asset API `Image(.avatar)` not `Image("avatar")`.
- `onChange()`: 2- or 0-parameter variant only, never 1-parameter. No `GeometryReader` where a newer alternative works: `containerRelativeFrame()`, `visualEffect()`, `Layout`.
- `@Entry` macro for custom `EnvironmentValues`, `FocusValues`, `Transaction`, `ContainerValues` keys. `ObservableObject` unavoidable (e.g. Combine debouncer) ⇒ add `import Combine`; SwiftUI no longer re-exports it.
- `Text`: automatic grammar agreement, English/French/German/Portuguese/Spanish/Italian — `Text("^[\(n) person](inflect: true)")` · styled runs via `+` or interpolation — `Text("Hello \(Text("World").bold())")`, both preserve per-run styling.
- Fill + stroke a shape with two chained modifiers — no overlay (iOS 17+). `ForEach` over `enumerated()`: `ForEach(items.enumerated(), id: \.element.id)` — no array conversion. iOS 26+: native `WebView` (`import WebKit`), not a hand-wrapped `WKWebView`.
- Commands on a STANDARD macOS menu (View/Edit/File/…): `CommandGroup(before:/after: <CommandGroupPlacement>)`, never `CommandMenu(name)` — `CommandMenu` always creates a NEW top-level menu, so `CommandMenu("View")` produces TWO View menus. View's built-in groups: `.sidebar` (Show/Hide Sidebar, Enter/Exit Full Screen), `.toolbar` (Show/Hide Toolbar); `.sidebar` exists even with no sidebar. `CommandMenu` only for genuinely new top-level menus.

*Soft-deprecated patterns: `references/soft-deprecation.md` + `references/soft-deprecated-apis.md`.*

## §Views

- Separate `View` structs, not computed properties or `@ViewBuilder` methods returning `some View` — computed properties share the parent's invalidation boundary; dedicated structs own their `@State`, lifecycle, `#Preview`.
- Flag excessively long `body` — extract subviews. Extract Button actions into methods; no business logic inline in `task()`, `onAppear()`, or elsewhere in `body`.
- `Button` hit-tests only its **rendered content** — `.padding()` / `.frame(maxWidth: .infinity)` around a short label are transparent dead zones, so clicks land on the glyphs and nowhere else. Put `.contentShape(.rect)` **inside** the Button's label, *after* frame+padding; outside the Button it decorates the wrapper and does nothing for hit-testing.
- Never `.onTapGesture` on a `Button`'s label — it competes with the Button and swallows the click it should receive; a Button needs no gesture help. `.focusable(false)` likewise suppresses interaction. Double-click: `.simultaneousGesture(TapGesture(count: 2))`, which doesn't consume the primary click.
- Async work tied to a view's lifetime: `.task` / `.task(id:)` — SwiftUI cancels automatically when the view leaves the view graph; never store a `Task` property and cancel by hand.
- `.task { }` on a conditional branch is cancelled when `@Observable` state changes swap branches. Attach lifecycle tasks to the always-present outer container; key re-firing with `.task(id:)` on `@State`.
- `TextField` with `axis: .vertical` over `TextEditor`, unless full-screen editing is required. `#Preview`, not the legacy `PreviewProvider` protocol. Rendering to images: `ImageRenderer`, not `UIGraphicsImageRenderer`.
- `TabView(selection:)`: bind an enum, not an integer or string. `Tab(_:systemImage:value:content:)` requires a non-optional `selection` binding (iOS 18+ / macOS 15+).
- Never `animation(_ animation: Animation?)` — always a value: `.animation(.bouncy, value: score)`. Chain with `withAnimation { } completion: { withAnimation { } }`, not multiple `withAnimation` calls with delays. `@Animatable` macro over manual `animatableData`.

*View factoring, invalidation boundaries, init costs, single-child `Group` anti-pattern: `references/structure.md`. `ForEach`/`List`/`Table` identity + collection performance: `references/foreach.md`. Building a container that takes caller-supplied content — API contract, which Apple layer to reach for, and the research method that lands on it: `references/custom-containers.md`.*

## §Data Flow

- Keep body code and logic separate — extract logic into `@Observable` classes. `@Observable` classes must be marked `@MainActor` unless the project has Main Actor default actor isolation.
- `@Observable` + `@State` (ownership) + `@Bindable` / `@Environment` (passing). Avoid `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject` unless unavoidable.
- "Stale value / didn't update" bugs are almost always state-consistency problems — an `@State` mirror drifted from source of truth. Fix by removing the desyncable mirror, not by swapping `.task(id:)` and `.onChange`.
- `@State` is `private`, owned by the view that created it. Never `@AppStorage` inside an `@Observable` class — it will not trigger view updates.
- Child both reads and writes parent state ⇒ pass `@Binding`, not an `onChange`/callback closure (closures are for one-shot actions with no parent state to mutate). `$`-prefixed projected bindings over inline `Binding(get:set:)` in a body.
- Child mutates one element of a collection ⇒ `ForEach($items) { $item in }` / `List($items)` — `Binding` is itself a collection whose elements are element bindings. Never build a keyed side-store to get a two-way handle the framework already vends.
- Numeric `TextField`: bind `Int`/`Double` via the `format:` initializer + `.keyboardType(.numberPad)` / `.keyboardType(.decimalPad)`. `Identifiable` conformance over `id: \.someProperty`.
- macOS: `@Environment(\.dismissWindow)` (macOS 14+) over `NSApp.keyWindow?.close()`.

*Deep `@Observable` per-property tracking, collection granularity, `@Binding` KeyPath patterns, `onChange` isolation: `references/dataflow.md`.*

## §Navigation

- `NavigationStack` or `NavigationSplitView`; flag all use of deprecated `NavigationView`.
- `navigationDestination(for:)` over `NavigationLink(destination:)` — flag the old pattern; never mix the two in one hierarchy; register once per data type.
- Presentation: attach `confirmationDialog()` to the UI that triggers it — Liquid Glass animations must originate from the correct source · alert with a single dismiss "OK" button and no action ⇒ omit the button entirely · optional data ⇒ `sheet(item:)` over `sheet(isPresented:)`, and `sheet(item: $item, content: SomeView.init)` when the item is the view's only init parameter.

## §Scenes & Windows

- `@State` lifetime follows the view's **structural-identity node**, and sheet / navigation / window / iPad-scene teardown each differ — see `references/scenes.md`.

## §Design

- Standard fonts, sizes, colors, spacing, padding, rounding, timing into a shared enum of constants, for uniformity. Avoid hard-coded padding and stack spacing unless requested.
- Never `UIScreen.main.bounds`; use `containerRelativeFrame()`, `visualEffect()`, or (last resort) `GeometryReader`. Avoid fixed frames unless content fits neatly — they break across device sizes and Dynamic Type.
- Minimum tap area on iOS is 44×44 — below that, targets are unreliable for anyone without precise motor control.
- `ContentUnavailableView` when data is missing or empty; with `searchable()` use `ContentUnavailableView.search` (not `.search(text:)`) for empty results.
- `Label` over `HStack` for icon + text side by side. System hierarchical styles (secondary/tertiary) over manual opacity. No `UIColor` — SwiftUI `Color` or asset-catalog colors.
- In `Form`, wrap `Slider` in `LabeledContent` for correct layout. `LabeledContent` also works outside `Form` for title-value displays; a custom `LabeledContentStyle` keeps layout consistent across views.
- `RoundedRectangle`: `.continuous` is the default — don't specify it. `bold()` over `fontWeight(.bold)`; `fontWeight()` only for non-bold weights with a specific reason. `.caption2` is extremely small — use sparingly; `.caption` is also small — use carefully.

## §Accessibility

- Respect user accessibility settings for fonts, colors, animations. "Reduce Motion" on ⇒ replace motion-based animations with opacity.
- Don't force font sizes; use Dynamic Type. Custom size needed: `@ScaledMetric` for iOS ≤ 18; `.font(.body.scaled(by:))` also available on iOS 26+.
- Flag images with unclear VoiceOver readings. Decorative: `Image(decorative:)` or `.accessibilityHidden(true)`. Informative: `.accessibilityLabel()`.
- Buttons with image labels must always include text, even if visually hidden — same for `Menu`: `Menu("Options", systemImage: "ellipsis.circle") { }` over image-only. Frequently changing button labels: recommend `accessibilityInputLabels()`.
- Color as an important differentiator ⇒ respect `.accessibilityDifferentiateWithoutColor`. Never `onTapGesture()` unless tap location or count is needed — use `Button`; if unavoidable, add `.accessibilityAddTraits(.isButton)`.

## §Performance

- Toggling modifier values: ternary expressions over `if/else` branching, preserving structural identity. Avoid `AnyView` unless absolutely required; use `@ViewBuilder`, `Group`, or generics.
- Dedicated `View` structs — computed properties create no new invalidation boundary.
- Keep view initializers minimal; move non-trivial work into `.task()`, which beats `onAppear()` for async work — cancelled automatically on disappear.
- Assume `body` runs frequently — keep sorting/filtering out of `body`, expensive inline transforms out of `List`/`ForEach` initializers. Derive transformed data with `let`, or cache in `@State` with explicit invalidation logic.
- Don't store `DateFormatter` etc. as properties; use `Text(date, format: .dateTime…)`. Don't store escaping `@ViewBuilder` closures on views; store the built view result.
- `ScrollView` with an opaque static background: `scrollContentBackground(.visible)`. Large data sets in `ScrollView`: `LazyVStack`/`LazyHStack`.

## §Swift

- Modern replacements: `replacing("a", with: "b")` not `replacingOccurrences(of:with:)` · `URL.documentsDirectory`, `appending(path:)` · `FormatStyle` not C-style `String(format: "%.2f", value)` · `Date.now` not `Date()` · `count(where:)` not `filter { }.count` · display years `"y"` not `"yyyy"` · parse via `Date(myString, strategy: .iso8601)` · `if let value {` not `if let value = value {`.
- Static member lookup: `.circle` over `Circle()`, `.borderedProminent` over `BorderedProminentButtonStyle()`. Omit `return` for single-expression functions; use `if`/`switch` as expressions.
- Avoid force unwraps and force `try` unless truly unrecoverable; prefer `fatalError()` with a description. Flag errors triggered by user actions that are swallowed silently.
- Filtering user input: `localizedStandardContains()`, not `contains()` or `localizedCaseInsensitiveContains()`. Person names: `PersonNameComponents` with modern formatting. Repeated sort closures: centralize via `Comparable`.
- `Double` over `CGFloat` except with optionals or `inout`. `import SwiftUI` already imports `UIKit`/`AppKit` — no extra import for `UIImage`/`NSImage`.
- Concurrency: `async`/`await` over closure-based variants. Never `DispatchQueue`. Never `Task.sleep(nanoseconds:)` — use `Task.sleep(for:)`. `Task.detached()` is often a bad idea — check any usage carefully. Flag mutable shared state not protected by an actor or `@MainActor`; strict concurrency — flag `@Sendable` violations and data races.

## §Hygiene

- Never commit secrets (API keys etc.). Never `@AppStorage` for usernames, passwords, or sensitive data — use Keychain.
- Comments and doc comments where logic is not self-evident. Unit tests for core logic; UI tests only where unit tests are impossible.
- SwiftLint configured ⇒ must return no warnings or errors. `Localizable.xcstrings` in use ⇒ prefer symbol keys with `extractionState: manual`.
- Xcode MCP configured ⇒ prefer its tools (`RenderPreview`, `DocumentationSearch`) over generic alternatives.

## References

Load on demand — read the file for your topic. The list below is the complete SwiftUI surface, so there is no second SwiftUI skill to reach for.

- `references/structure.md` — separate `View` struct vs computed property / `@ViewBuilder` method, `init` cost, single-child `Group` anti-pattern, extract-for-testability; also performance.md.
- `references/custom-containers.md` — building a container that takes caller-supplied content: the L0–L5 layer ladder (`Group` collect → data-driven `Content == ForEach<…>` → `Content: DynamicViewContent` bound → value builder → named row modifiers → `Group(subviews:)` decompose → `Layout`), pinning `Content`, caller-applied row modifiers and their apply order, `.tag` is write-only for custom containers, `Binding<V?>` vs `Binding<V>?`, why nothing fills a scroll axis, drag-reorder across the version floor, and which tool answers purpose vs mechanism vs availability.
- `references/availability-gating.md` — adopting a newer API without dropping the deployment floor: which tool prints `@available` (`goToDefinition`, not hover), mirroring the API's own platform list, the five gating shapes (branch in one modifier · twin types + chooser · compile-time `#if` for an SDK-absent symbol · gated previews · version-free value + gated conversion `init`, when the newer *type* would go viral through stored properties), deprecating your own fallback so the compiler schedules its removal.
- `references/progressive-disclosure.md` — designing the API of a component you're writing: call-site vs declaration-site perspective, the four strategies (common cases, intelligent defaults, optimise the call site, compose don't enumerate), the two overload questions, `Table`'s simplification chain, `Spacer`-over-`arrangement`-enum; WWDC22 principles, 2022-era spellings.
- `references/dataflow.md` — `@Observable`/`@State`/`@Binding`, per-property tracking, collection granularity, `.onChange` isolation, KeyPath bindings, `Binding` as an upward *intent* channel (child writes a request value, container applies in `.task(id:)` and consumes it) + why a pile of callback closures is a delegate, numeric `TextField` `format:`, "stale value / didn't update" desynced `@State` mirror, SwiftData `@Query`/`modelContext`; SwiftData model layer → swiftdata-pro.
- `references/environment.md` — `@Environment`, `EnvironmentKey`/`EnvironmentValues`, `FocusedValue`, `@Entry`, comparison perf; propagation across sheets → scenes.md.
- `references/foreach.md` — `ForEach`/`List`/`Table`/`OutlineGroup` element identity, index-as-id and transient-id anti-patterns, identity-driven row diffing; scroll/lazy perf → performance.md.
- `references/modifiers.md` — conditional (`.if`) view modifier anti-pattern. `references/animations.md` — custom `Animatable` types, `@Animatable` macro, `animatableData`.
- `references/localization.md` — `LocalizedStringKey`, `LocalizedStringResource` vs `String`, `bundle: #bundle`, format styles, RTL, runtime case transforms, translator comments.
- `references/soft-deprecation.md` — how to *identify* a soft-deprecated API (`deprecated: 100000.0`), out-of-scope scoping rule, when to migrate; the list is `references/soft-deprecated-apis.md` — searchable *list* of soft-deprecated SwiftUI APIs + replacements (`NavigationView`, `foregroundColor`, …).
- `references/performance.md` — `_ConditionalContent` from `if/else`, `@ViewBuilder`-let, `LazyVStack`/`LazyHStack`, frequent-`body` recompute, escaping-closure storage.
- `references/swift.md` — modern Swift idioms (`count(where:)`, `Date.now`, `FormatStyle`), redundant `MainActor.run`; NOT deep concurrency — see swift-concurrency-pro.
- `references/accessibility.md` — Dynamic Type / `@ScaledMetric`, VoiceOver labels, Reduce Motion, `.labelStyle(.iconOnly)`, `accessibilityInputLabels`.
- `references/scenes.md` — scene/window state *lifetime* + teardown: `@State` resets via `.id()` / branch flip, `.task` view-scoped vs outliving the view, `sheet(item:)`/`fullScreenCover` teardown, `navigationDestination` registration scope, macOS window close + `Settings` won't-quit trap, iPad `@SceneStorage`; *correctness* → dataflow.md.
- Deployment target 27+ only, NOT for pre-27 targets: `references/state-macro.md` — `@State` property-wrapper → macro migration, init-assignment incompatibilities · `references/content-builder.md` — `@ContentBuilder` unification of `@ViewBuilder`, ambiguous `overlay`/`background` ShapeStyle errors · `references/deprecations.md` — APIs *hard*-deprecated in SDK 27.0 (e.g. `statusBarHidden` on visionOS), NOT soft-deprecations.

## §Output

**Reviewing.** Report everything you found, then rank it — a separate filtering pass is the user's, not yours to pre-empt, and a reviewer told to be selective reports less than it saw. Group findings by file; per finding give the line(s), the rule or idiom at stake, and a brief before/after. Say which files you read and found clean, so the reader can tell coverage from silence. Close with a severity-ordered summary:

`**Critical | Important | Suggestion:** description — file:line`

Severity is about consequence — Critical is a crash, data loss, a shipped-wrong-behaviour bug or a broken accessibility path; Important is a defect users will hit or a correctness risk in identity/data flow; Suggestion is everything else worth saying. A hand-composed shape where a primitive exists, and unnecessary view reloading, are Important or higher even though the code works.

**Authoring.** Report the change, not a review of it: what you edited, which idiom or primitive you chose and why, any availability gate you added, and anything you left alone that a reviewer would reasonably ask about.
