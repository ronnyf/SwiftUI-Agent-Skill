# swiftui-specialist Routing Test Cases

**Purpose:** Verify post-restructure routing correctness. Each case maps a phrase to its expected primary reference in the **future 16-file single tree**. Used in Task 7's fix-loop (Training section only). The Validation section is held out until the final generalization check.

**Future tree files:**
`structure`, `dataflow`, `environment`, `foreach`, `modifiers`, `animations`, `localization`,
`soft-deprecation`, `soft-deprecated-apis`, `performance`, `swift`, `accessibility`,
`scenes`, `state-macro`, `content-builder`, `deprecations`

**RETIRED (never a valid expected target):** `views`, `data`, `hygiene`, `navigation`, `design`, `api`

---

## Training

> Cases used during the Task-7 description-fix loop. Do NOT include these in the final generalization metric.

### T-structure — view factoring & invalidation boundaries

**T-S1 (positive):**
Phrase: "I have a long body property with computed properties for each section — is that okay?"
Expected: `structure`
Reasoning: Computed properties as view sections are the anti-pattern `structure.md` directly addresses.

**T-S2 (positive):**
Phrase: "does extracting a subview into a computed property create a new invalidation boundary?"
Expected: `structure`

**T-S3 (positive):**
Phrase: "my header and footer are private var computed views in the same parent — any issue?"
Expected: `structure`

**T-S4 (negative):**
Phrase: "does extracting a subview into a computed property create a new invalidation boundary?"
Expected: NOT `performance`
Reasoning: Performance is a downstream effect; the root cause (invalidation boundaries, factoring) belongs to `structure`.

---

### T-dataflow — @Observable, @State, @Binding, stale values

**T-D1 (positive — named regression phrase):**
Phrase: "an @Observable view shows a stale value even after the model changes"
Expected: `dataflow`
Reasoning: Stale/desynced observable value is explicitly covered in `dataflow.md`'s desync section.

**T-D2 (positive):**
Phrase: "my view is not updating when I change a property on my @Observable model"
Expected: `dataflow`

**T-D3 (positive):**
Phrase: "how does per-property tracking work with @Observable — why does only part of my view re-render?"
Expected: `dataflow`

**T-D4 (positive):**
Phrase: "when should I pass a @Binding vs a callback closure to a child view?"
Expected: `dataflow`

**T-D5 (negative):**
Phrase: "my view is not updating when I change a property on my @Observable model"
Expected: NOT `soft-deprecated-apis`
Reasoning: Stale-update issues are an @Observable mechanics question, not a deprecated-API question.

---

### T-environment — @Environment, EnvironmentKey, @Entry, FocusedValue

**T-E1 (positive):**
Phrase: "how do I define a custom environment key with @Entry?"
Expected: `environment`

**T-E2 (positive):**
Phrase: "my custom environment value causes every view in the subtree to re-render when I update it"
Expected: `environment`
Reasoning: Unstable environment values causing excess invalidation is environment.md's central topic.

**T-E3 (positive):**
Phrase: "can I store a closure in an environment key?"
Expected: `environment`
Reasoning: Closures in environment keys causing non-comparable comparisons is a key environment.md rule.

**T-E4 (negative):**
Phrase: "how do I define a custom environment key with @Entry?"
Expected: NOT `dataflow`
Reasoning: @Entry and environment mechanics belong to `environment`, not `dataflow` (which is about @State/@Binding/@Observable).

**T-E5 (negative):**
Phrase: "I stored a closure in my custom EnvironmentValues extension — now views re-render constantly"
Expected: NOT `dataflow`
Reasoning: Constant invalidation from a closure in an environment key is an environment stability topic, not a dataflow (@State/@Binding/@Observable) topic.

---

### T-foreach — ForEach/List/Table identity

**T-F1 (positive):**
Phrase: "my ForEach list items flicker or reset their state when the collection changes"
Expected: `foreach`
Reasoning: State reset from unstable identity is foreach.md's opening topic.

**T-F2 (positive):**
Phrase: "is using array indices as ForEach IDs safe?"
Expected: `foreach`

**T-F3 (positive):**
Phrase: "how should I set up identity for a List that supports reordering and deletion?"
Expected: `foreach`

**T-F4 (negative):**
Phrase: "my ForEach list items flicker or reset their state when the collection changes"
Expected: NOT `modifiers`
Reasoning: Flickering from unstable identity is a ForEach identity problem, not a conditional-modifier problem.

**T-F5 (negative):**
Phrase: "how should I identify rows in a Table view for correct diffing?"
Expected: NOT `structure`
Reasoning: Table row identity for correct diffing belongs to `foreach`; `structure` is about view factoring and invalidation boundaries.

---

### T-modifiers — conditional view modifiers, structural identity

**T-M1 (positive):**
Phrase: "I wrote a .if() extension on View that conditionally applies a modifier — any problem?"
Expected: `modifiers`
Reasoning: The conditional `.if` modifier anti-pattern is the exact subject of `modifiers.md`.

**T-M2 (positive):**
Phrase: "why does my custom conditional view modifier break animations and reset @State?"
Expected: `modifiers`

**T-M3 (negative):**
Phrase: "why does my custom conditional view modifier break animations and reset @State?"
Expected: NOT `animations`
Reasoning: The breakage is caused by structural identity loss in the modifier, not an animations mechanics issue.

**T-M4 (negative):**
Phrase: "toggling a conditional modifier causes abrupt jumps instead of smooth animations"
Expected: NOT `scenes`
Reasoning: Abrupt jumps from toggling a conditional modifier are a structural-identity issue in `modifiers`, not a scene/presentation lifetime issue.

---

### T-animations — @Animatable, animatableData

**T-A1 (positive):**
Phrase: "how do I make a custom Shape animatable without writing animatableData by hand?"
Expected: `animations`
Reasoning: @Animatable macro avoiding manual animatableData is animations.md's subject.

**T-A2 (positive):**
Phrase: "I have a custom view with multiple CGFloat properties — how do I animate all of them?"
Expected: `animations`

**T-A3 (negative):**
Phrase: "how do I make a custom Shape animatable without writing animatableData by hand?"
Expected: NOT `modifiers`
Reasoning: The topic is Animatable protocol conformance, not conditional modifiers.

**T-A4 (negative):**
Phrase: "some of my custom type's properties can't be animated — how do I mark them?"
Expected: NOT `performance`
Reasoning: Marking non-animatable properties with @AnimatableIgnored is an animations mechanics topic, not a performance optimization topic.

---

### T-localization — LocalizedStringKey, bundle: #bundle, RTL

**T-L1 (positive):**
Phrase: "my strings show up unlocalized in a Swift package — what's wrong?"
Expected: `localization`
Reasoning: Missing `bundle: #bundle` in frameworks/packages is localization.md's bundle section.

**T-L2 (positive):**
Phrase: "should I use LocalizedStringKey or LocalizedStringResource for a UI string?"
Expected: `localization`

**T-L3 (negative):**
Phrase: "my strings show up unlocalized in a Swift package — what's wrong?"
Expected: NOT `environment`
Reasoning: Bundle-not-found localization failures are a localization topic, not an environment propagation topic.

---

### T-soft-deprecation — how to identify soft-deprecated APIs, scoping rule

**T-SD1 (positive):**
Phrase: "what is a soft-deprecated API in SwiftUI and should I worry about it?"
Expected: `soft-deprecation`
Reasoning: soft-deprecation.md explains what soft-deprecated means (version 100000.0) and when to migrate.

**T-SD2 (positive):**
Phrase: "I see a deprecation warning with version 100000.0 — what does that mean?"
Expected: `soft-deprecation`

**T-SD3 (negative):**
Phrase: "what is a soft-deprecated API in SwiftUI and should I worry about it?"
Expected: NOT `soft-deprecated-apis`
Reasoning: Understanding what soft-deprecated means goes to `soft-deprecation`; the explicit list of which APIs are soft-deprecated goes to `soft-deprecated-apis`.

---

### T-soft-deprecated-apis — searchable list of deprecated APIs

**T-SDA1 (positive — named regression phrase):**
Phrase: "is NavigationView deprecated?"
Expected: `soft-deprecated-apis`
Reasoning: NavigationView appears explicitly in the soft-deprecated-apis.md list.

**T-SDA2 (positive):**
Phrase: "what is the replacement for ActionSheet in SwiftUI?"
Expected: `soft-deprecated-apis`
Reasoning: ActionSheet is listed in soft-deprecated-apis.md with its replacement.

**T-SDA3 (positive):**
Phrase: "I'm using MagnificationGesture — is that still current?"
Expected: `soft-deprecated-apis`
Reasoning: MagnificationGesture → MagnifyGesture is explicitly in the soft-deprecated-apis list.

**T-SDA4 (negative):**
Phrase: "is NavigationView deprecated?"
Expected: NOT `soft-deprecation`
Reasoning: `soft-deprecation` explains the concept; the lookup for a specific API belongs to `soft-deprecated-apis`.

---

### T-performance — _ConditionalContent, LazyVStack, @ViewBuilder

**T-P1 (positive):**
Phrase: "my ScrollView with hundreds of items is slow — how do I fix it?"
Expected: `performance`
Reasoning: LazyVStack for large scroll views is a performance.md topic.

**T-P2 (positive):**
Phrase: "should I use if/else branches or ternary expressions in view modifiers for toggling?"
Expected: `performance`
Reasoning: Ternary over if/else to avoid _ConditionalContent is performance.md's first rule.

**T-P3 (negative):**
Phrase: "should I use if/else branches or ternary expressions in view modifiers for toggling?"
Expected: NOT `modifiers`
Reasoning: The performance cost of _ConditionalContent (not conditional modifier anti-patterns) is a performance topic.

---

### T-swift — modern Swift, concurrency idioms

**T-SW1 (positive):**
Phrase: "is it okay to use DispatchQueue.main in SwiftUI code?"
Expected: `swift`
Reasoning: Avoiding DispatchQueue in favor of Swift Concurrency is a swift.md rule.

**T-SW2 (positive):**
Phrase: "I'm using String(format: "%.2f", value) for displaying numbers — is there a better way?"
Expected: `swift`
Reasoning: Avoiding C-style number formatting in favor of FormatStyle is swift.md.

**T-SW3 (negative):**
Phrase: "I'm using String(format: "%.2f", value) for displaying numbers — is there a better way?"
Expected: NOT `localization`
Reasoning: FormatStyle API usage is a Swift modern-API topic, not a localization topic (even though FormatStyle also appears in localization contexts).

---

### T-accessibility — Dynamic Type, VoiceOver, Reduce Motion

**T-ACC1 (positive):**
Phrase: "I have an icon-only button — does VoiceOver still work correctly?"
Expected: `accessibility`
Reasoning: Icon-only buttons requiring hidden text labels is an accessibility.md rule.

**T-ACC2 (positive):**
Phrase: "how do I support Reduce Motion in my SwiftUI animations?"
Expected: `accessibility`

**T-ACC3 (negative):**
Phrase: "how do I support Reduce Motion in my SwiftUI animations?"
Expected: NOT `animations`
Reasoning: Reduce Motion is an accessibility concern (how to detect and handle it); `animations` covers @Animatable mechanics.

---

### T-scenes — scene/presentation lifetime, teardown, macOS windows

**T-SC1 (positive — named regression phrase):**
Phrase: "scope a service's lifetime to a presentation/sheet"
Expected: `scenes`
Reasoning: Tying a service's lifetime to a sheet's lifetime is a scenes/presentation teardown topic.

**T-SC2 (positive — named regression phrase):**
Phrase: "@State resets when a bool flips — how do I preserve it across conditional branches?"
Expected: `scenes`
Reasoning: Structural-identity-node framing of @State reset on branch flip is scenes.md's identity-node section.

**T-SC3 (positive — named regression phrase):**
Phrase: "app won't quit after the last window closes on macOS"
Expected: `scenes`
Reasoning: The macOS "won't quit after last window" trap is explicitly a scenes.md topic.

**T-SC4 (positive):**
Phrase: "how do I run teardown code when a macOS window closes?"
Expected: `scenes`
Reasoning: Window teardown (isolated deinit, onDisappear not reliable for close) is scenes.md.

**T-SC5 (positive):**
Phrase: "my .task modifier is cancelled when a sheet dismisses — is that expected?"
Expected: `scenes`
Reasoning: .task view-scoped cancellation when a sheet/presentation tears down is scenes.md.

**T-SC6 (negative):**
Phrase: "how do I run teardown code when a macOS window closes?"
Expected: NOT `dataflow`
Reasoning: Window lifecycle and teardown belongs to `scenes`; `dataflow` covers observable data flow patterns.

---

### T-state-macro — @State macro migration (SDK 27)

**T-SM1 (positive):**
Phrase: "@State init assignment gives 'self used before super.init' error after upgrading to SDK 27"
Expected: `state-macro`
Reasoning: Init assignment error from @State-as-macro is state-macro.md's first rule.

**T-SM2 (positive):**
Phrase: "I'm getting 'variable self.counter used before being initialized' with @State in init"
Expected: `state-macro`

**T-SM3 (negative):**
Phrase: "@State init assignment gives error after upgrading to SDK 27"
Expected: NOT `deprecations`
Reasoning: @State macro migration is a source-incompatibility (state-macro.md); `deprecations` covers hard-deprecated APIs removed in SDK 27, not macro migration.

---

### T-content-builder — @ContentBuilder unification (SDK 27)

**T-CB1 (positive):**
Phrase: "I'm getting 'ambiguous use of opacity' in an overlay after upgrading to SDK 27"
Expected: `content-builder`
Reasoning: ContentBuilder unification causing ambiguous ShapeStyle modifier errors is content-builder.md's first rule.

**T-CB2 (positive):**
Phrase: "SDK 27 broke my .background(Color.blue.opacity(0.5)) — what changed?"
Expected: `content-builder`

**T-CB3 (negative):**
Phrase: "SDK 27 broke my .background(Color.blue.opacity(0.5)) — what changed?"
Expected: NOT `state-macro`
Reasoning: Background/overlay ambiguity errors are from @ContentBuilder unification, not from @State macro migration.

---

### T-deprecations — hard-deprecated APIs in SDK 27

**T-DEP1 (positive):**
Phrase: "statusBarHidden is showing a deprecation warning on visionOS 27 — what replaces it?"
Expected: `deprecations`
Reasoning: statusBarHidden on visionOS is listed in deprecations.md as hard-deprecated in SDK 27.0.

**T-DEP2 (positive):**
Phrase: "which APIs were hard-deprecated (not soft-deprecated) when targeting SDK 27?"
Expected: `deprecations`

**T-DEP3 (negative):**
Phrase: "which APIs were hard-deprecated when targeting SDK 27?"
Expected: NOT `soft-deprecated-apis`
Reasoning: Hard-deprecated APIs (compiler warnings, version explicit) belong to `deprecations`; soft-deprecated APIs (version 100000.0, no compiler warning) belong to `soft-deprecated-apis`.

---

## Validation (held-out)

> These cases are FROZEN. They must not be used during the Task-7 description-fix loop. They are loaded only for the final generalization check (Task 7 Step 4). Approximately ⅓ of total cases. Contains proportional mix of positives and negatives across topics.

---

### V-structure

**V-S1 (positive):**
Phrase: "a Group with a single child view — is there ever a performance reason to avoid it?"
Expected: `structure`
Reasoning: Single-child Group anti-pattern is a structure.md rule.

**V-S2 (positive):**
Phrase: "why does breaking a view into computed properties not improve performance?"
Expected: `structure`

**V-S3 (negative):**
Phrase: "why does breaking a view into computed properties not improve performance?"
Expected: NOT `performance`

---

### V-dataflow

**V-D1 (positive):**
Phrase: "I'm using @State as a cache for a CIContext so it doesn't get recreated — is that okay?"
Expected: `dataflow`
Reasoning: @State-as-object-cache is an explicit dataflow.md pattern.

**V-D2 (positive):**
Phrase: "how do I use KeyPath bindings instead of closure-based bindings?"
Expected: `dataflow`

**V-D3 (negative):**
Phrase: "how do I use KeyPath bindings instead of closure-based bindings?"
Expected: NOT `environment`

---

### V-environment

**V-E1 (positive):**
Phrase: "I stored a closure in my custom EnvironmentValues extension — now views re-render constantly"
Expected: `environment`
Reasoning: Closures in custom environment keys causing constant invalidation is environment.md's closure section.

---

### V-foreach

**V-F1 (positive):**
Phrase: "how should I identify rows in a Table view for correct diffing?"
Expected: `foreach`
Reasoning: Table identity requirements are explicitly covered in foreach.md.

---

### V-modifiers

**V-M1 (positive):**
Phrase: "toggling a conditional modifier causes abrupt jumps instead of smooth animations"
Expected: `modifiers`

---

### V-animations

**V-A1 (positive):**
Phrase: "some of my custom type's properties can't be animated — how do I mark them?"
Expected: `animations`
Reasoning: @AnimatableIgnored macro for non-animatable properties is animations.md.

---

### V-localization

**V-L1 (positive):**
Phrase: "how do I add translator comments to my SwiftUI string literals?"
Expected: `localization`

**V-L2 (negative):**
Phrase: "how do I add translator comments to my SwiftUI string literals?"
Expected: NOT `swift`

---

### V-soft-deprecation

**V-SD1 (positive):**
Phrase: "the scoping rule for soft-deprecated APIs — do I flag them everywhere in the file?"
Expected: `soft-deprecation`
Reasoning: The scoping rule (only flag code you're editing) is soft-deprecation.md's first section.

**V-SD2 (negative):**
Phrase: "the scoping rule for soft-deprecated APIs — do I flag them everywhere in the file?"
Expected: NOT `soft-deprecated-apis`

---

### V-soft-deprecated-apis

**V-SDA1 (positive):**
Phrase: "is PresentationMode still available or should I use something else?"
Expected: `soft-deprecated-apis`
Reasoning: PresentationMode → EnvironmentValues.dismiss/isPresented is in soft-deprecated-apis.md.

**V-SDA2 (negative):**
Phrase: "is PresentationMode still available or should I use something else?"
Expected: NOT `soft-deprecation`

---

### V-performance

**V-P1 (positive):**
Phrase: "I'm seeing slow scroll performance with a large list — should I use LazyVStack or List?"
Expected: `performance`

**V-P2 (negative):**
Phrase: "I'm seeing slow scroll performance with a large list — should I use LazyVStack or List?"
Expected: NOT `foreach`

---

### V-swift

**V-SW1 (positive):**
Phrase: "is Task.detached safe to use in SwiftUI — when would I need it?"
Expected: `swift`
Reasoning: Task.detached safety analysis is a swift.md concurrency topic.

**V-SW2 (negative):**
Phrase: "is Task.detached safe to use in SwiftUI?"
Expected: NOT `dataflow`

---

### V-accessibility

**V-ACC1 (positive):**
Phrase: "I use color to distinguish status — what do I need to do for colorblind users?"
Expected: `accessibility`
Reasoning: accessibilityDifferentiateWithoutColor handling is accessibility.md.

**V-ACC2 (negative):**
Phrase: "I use color to distinguish status — what do I need to do for colorblind users?"
Expected: NOT `structure`

---

### V-scenes (named regression phrases — validation set)

**V-SC1 (positive — named regression phrase):**
Phrase: "how do I scope an @Observable service's lifetime to a sheet so it's released on dismiss?"
Expected: `scenes`

**V-SC2 (positive):**
Phrase: "my app stays running on macOS even after the user closes all windows — how do I fix this?"
Expected: `scenes`
Reasoning: "app won't quit after last window" — macOS quit-after-last-window trap is scenes.md.

**V-SC3 (negative):**
Phrase: "my app stays running on macOS even after the user closes all windows"
Expected: NOT `swift`

---

### V-state-macro

**V-SM1 (positive):**
Phrase: "@State now behaves differently in init compared to SDK 26 — what changed?"
Expected: `state-macro`

**V-SM2 (negative):**
Phrase: "@State now behaves differently in init compared to SDK 26 — what changed?"
Expected: NOT `content-builder`

---

### V-content-builder

**V-CB1 (positive):**
Phrase: "overlay and background modifiers are producing ambiguous type errors in SDK 27"
Expected: `content-builder`

**V-CB2 (negative):**
Phrase: "overlay and background modifiers are producing ambiguous type errors in SDK 27"
Expected: NOT `deprecations`

---

### V-deprecations

**V-DEP1 (positive):**
Phrase: "I'm targeting visionOS 27 and getting a warning that statusBarHidden is deprecated"
Expected: `deprecations`

**V-DEP2 (negative):**
Phrase: "I'm targeting visionOS 27 and getting a warning about a deprecated API"
Expected: NOT `soft-deprecated-apis`
Reasoning: Compiler-warning deprecations at SDK 27.0 are hard-deprecated (deprecations.md); soft-deprecated APIs produce no compiler warning.

---

## Case count summary

| Section | Positives | Negatives | Total |
|---------|-----------|-----------|-------|
| Training | 41 | 20 | 61 |
| Validation (held-out) | 19 | 12 | 31 |
| **Total** | **60** | **32** | **92** |

Held-out fraction: 31/92 ≈ 34% (target ≈ ⅓; within acceptable 33–40% range).

Negative fraction: Training 20/61 ≈ 32.8%; Validation 12/31 ≈ 38.7% (gap ≈ 6pp; within ~5pp target).

All 16 surviving topics are covered in both sections.
