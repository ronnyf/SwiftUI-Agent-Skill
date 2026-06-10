---
name: swiftui-pro
description: Comprehensively reviews SwiftUI code for best practices on modern APIs, maintainability, and performance. Use when reading, writing, or reviewing SwiftUI projects.
license: MIT
metadata:
  author: Paul Hudson
  version: "1.2"
---

Review Swift and SwiftUI code for correctness, modern API usage, and adherence to project conventions. Report only genuine problems - do not nitpick or invent issues.

Review process:

1. Check for hard-deprecated API using `references/api.md`. Also check soft-deprecated patterns using `references/soft-deprecation.md` and `references/soft-deprecated-apis.md`.
1. Check that views are structured correctly (separate `View` types, not computed properties) using `references/structure.md`. Check view composition, modifiers, and animations using `references/views.md`, `references/modifiers.md`, and `references/animations.md`.
1. Validate that data flow is configured correctly using `references/data.md`. For `@Observable`, `@Binding`, `ForEach` identity, and `onChange` isolation apply the deep rules in `references/dataflow.md` and `references/foreach.md`.
1. Ensure navigation is updated and performant using `references/navigation.md`.
1. Ensure the code uses designs that are accessible and compliant with Apple's Human Interface Guidelines using `references/design.md`.
1. Validate accessibility compliance including Dynamic Type, VoiceOver, and Reduce Motion using `references/accessibility.md`.
1. Ensure the code is able to run efficiently using `references/performance.md`.
1. Check environment values and `@Entry` usage using `references/environment.md`.
1. Validate localization — user-facing text, `LocalizedStringResource`, format styles — using `references/localization.md`.
1. Quick validation of Swift code using `references/swift.md`.
1. Final code hygiene check using `references/hygiene.md`.
1. **If the deployment target is iOS/macOS/watchOS/tvOS/visionOS 27 or later:** check for source incompatibilities from the `@State` macro migration (`references/state-macro.md`), `@ContentBuilder` unification (`references/content-builder.md`), and SDK 27 hard-deprecations (`references/deprecations.md`).

If doing a partial review, load only the relevant reference files.


## Core Instructions

- iOS 26 exists, and is the default deployment target for new apps.
- Target Swift 6.2 or later, using modern Swift concurrency.
- As a SwiftUI developer, the user will want to avoid UIKit unless requested.
- Do not introduce third-party frameworks without asking first.
- Break different types up into different Swift files rather than placing multiple structs, classes, or enums into a single file.
- Use a consistent project structure, with folder layout determined by app features.


## Output Format

Organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated (e.g., "Use `foregroundStyle()` instead of `foregroundColor()`").
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

Example output:

### ContentView.swift

**Line 12: Use `foregroundStyle()` instead of `foregroundColor()`.**

```swift
// Before
Text("Hello").foregroundColor(.red)

// After
Text("Hello").foregroundStyle(.red)
```

**Line 24: Icon-only button is bad for VoiceOver - add a text label.**

```swift
// Before
Button(action: addUser) {
    Image(systemName: "plus")
}

// After
Button("Add User", systemImage: "plus", action: addUser)
```

**Line 31: Avoid `Binding(get:set:)` in view body - use `@State` with `onChange()` instead.**

```swift
// Before
TextField("Username", text: Binding(
    get: { model.username },
    set: { model.username = $0; model.save() }
))

// After
TextField("Username", text: $model.username)
    .onChange(of: model.username) {
        model.save()
    }
```

### Summary

1. **Accessibility (high):** The add button on line 24 is invisible to VoiceOver.
2. **Deprecated API (medium):** `foregroundColor()` on line 12 should be `foregroundStyle()`.
3. **Data flow (medium):** The manual binding on line 31 is fragile and harder to maintain.

End of example.


## References

- `references/accessibility.md` - Dynamic Type, VoiceOver, Reduce Motion, and other accessibility requirements.
- `references/animations.md` - custom `Animatable` types.
- `references/api.md` - hard-deprecated APIs and their modern replacements.
- `references/content-builder.md` - SDK 27: `@ContentBuilder` unification and source incompatibilities.
- `references/data.md` - data flow, shared state, and property wrappers (overview).
- `references/dataflow.md` - deep `@Observable`, `@Binding`, per-property tracking granularity, `onChange` isolation, KeyPath bindings.
- `references/deprecations.md` - SDK 27: APIs hard-deprecated in 27.0.
- `references/design.md` - guidance for building accessible apps that meet Apple's Human Interface Guidelines.
- `references/environment.md` - `@Environment`, `@Entry`, `EnvironmentKey`, `FocusedValues`, and performance traps.
- `references/foreach.md` - `ForEach` / `List` / `Table` identity requirements, collection patterns, and performance.
- `references/hygiene.md` - making code compile cleanly and be maintainable in the long term.
- `references/localization.md` - `LocalizedStringKey`, `LocalizedStringResource`, format styles, RTL, translator comments.
- `references/modifiers.md` - view modifier usage, especially conditional modifiers.
- `references/navigation.md` - navigation using `NavigationStack`/`NavigationSplitView`, plus alerts, confirmation dialogs, and sheets.
- `references/performance.md` - optimizing SwiftUI code for maximum performance.
- `references/soft-deprecated-apis.md` - searchable list of all soft-deprecated SwiftUI APIs with replacements.
- `references/soft-deprecation.md` - how to identify soft-deprecated APIs and when to migrate.
- `references/state-macro.md` - SDK 27: `@State` migrated from property wrapper to macro — source incompatibilities and fixes.
- `references/structure.md` - separate `View` types vs computed properties, init costs, single-child `Group` anti-pattern.
- `references/swift.md` - tips on writing modern Swift code, including using Swift Concurrency effectively.
- `references/views.md` - view composition and animation.
