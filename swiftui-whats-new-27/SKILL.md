---
description: "New SwiftUI APIs, behaviors, and deprecations introduced in the 2027 OS releases (iOS 27, macOS 27, watchOS 27, tvOS 27, visionOS 27). Use when the user asks what's new in SwiftUI in iOS 27, macOS 27, watchOS 27, tvOS 27, or visionOS 27, when writing or updating SwiftUI code that touches new APIs or deprecated patterns, or when resolving source-incompatibilities introduced by the SDK 27.0 toolchain."
name: swiftui-whats-new-27
---
Use these references to understand what changed in SwiftUI for the 2027 OS releases. Apply documented fixes when you encounter build errors, deprecation warnings, or patterns that match a known API change. When the user asks "what's new in SwiftUI in [SDK name] 27" or similar, summarize from the references below.

# SDK 27.0

- `references/state-macro.md`: `@State` migrated from a property wrapper to a macro — source-incompatible init and composition issues.
- `references/content-builder.md`: Unified result builders under `@ContentBuilder` - source-incompatible in places that relied on the existing structure of result builders, and type-check performance regression in Swift Charts with deeply branching content.
- `references/deprecations.md`: APIs hard-deprecated in SDK 27.0 — `statusBarHidden` on visionOS (no effect, remove the call). Soft-deprecated APIs are covered by the `swiftui-specialist` skill.