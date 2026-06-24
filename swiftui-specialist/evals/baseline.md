# swiftui-specialist Routing Baseline

**Captured:** 2026-06-24 (Task 1 — before any restructure edits)
**Purpose:** Pre-change snapshot. Task 3's gate compares the restructured SKILL.md token count against this file's numbers. Task 7 verifies the regression phrases no longer return `none`/`ambiguous`.

---

## SKILL.md token count (pre-change)

| Metric | Value |
|--------|-------|
| `wc -w` (words) | 2664 |
| `wc -c` (bytes) | 20502 |
| Approx tokens (bytes ÷ 4) | ~5125 |

> Note: The spec predicted ~5,125 tokens; measured value matches.

---

## Current reference tree (21 files)

The current tree has a structural problem: it is a "dual tree" — Apple's idiomatic-pattern set (9 files) and Paul Hudson's review-checklist set (9 files) have heavily overlapping topics. Plus 3 SDK-27 migration files. Topics collide: data flow lives in both `data.md` AND `dataflow.md`; views/structure lives in both `views.md` AND `structure.md`.

**Apple idiomatic-pattern set (9):**
- `structure.md` — view factoring, invalidation boundaries, init cost, single-child Group anti-pattern
- `dataflow.md` — @Observable, @State, @Binding, per-property tracking, collection granularity, .onChange, KeyPath bindings
- `environment.md` — @Environment, EnvironmentKey, FocusedValue, @Entry, performance pitfalls
- `foreach.md` — ForEach/List/Table/OutlineGroup identity, row structure, List performance
- `modifiers.md` — conditional view modifiers, structural identity loss
- `animations.md` — @Animatable macro, animatableData
- `localization.md` — LocalizedStringKey, LocalizedStringResource, bundle: #bundle, format styles, RTL
- `soft-deprecation.md` — how to identify soft-deprecated APIs, scoping rule, when to migrate
- `soft-deprecated-apis.md` — searchable list of soft-deprecated SwiftUI APIs + replacements

**Paul Hudson review-checklist set (9):**
- `api.md` — modern API and deprecated replacements (overlaps §API inline + soft-deprecated-apis.md)
- `views.md` — view composition, animation, Tab API, .task on conditional branches (overlaps structure.md)
- `data.md` — data flow, property wrappers, stale-value bugs (overlaps dataflow.md)
- `navigation.md` — NavigationStack/NavigationSplitView, alerts, dialogs, sheets
- `design.md` — HIG, accessible design, UIScreen.main.bounds, tap targets
- `accessibility.md` — Dynamic Type, VoiceOver, Reduce Motion, icon-only buttons
- `performance.md` — _ConditionalContent, @ViewBuilder-let, LazyVStack, view struct factoring
- `swift.md` — modern Swift, concurrency idioms, MainActor, async/await
- `hygiene.md` — secrets, comments, tests, AppStorage, SwiftLint

**SDK-27 migration set (3):**
- `state-macro.md` — @State migrated from property wrapper to macro
- `content-builder.md` — @ContentBuilder unification
- `deprecations.md` — APIs hard-deprecated in SDK 27.0

**Orphaned inline section (no reference file):**
- §Scenes & Windows (macOS multi-window): per-window @State, onDisappear, isolated deinit — lives only as inline SKILL.md prose; no `scenes.md` reference file exists.

---

## Regression phrase routing (current tree)

Routing methodology: each phrase was evaluated independently twice against the current SKILL.md reference descriptions. A phrase is marked `ambiguous` when the two passes produced different primary targets, or when both passes named two equally-matching files.

| # | Phrase | Pass 1 | Pass 2 | Result |
|---|--------|--------|--------|--------|
| R1 | "scope a service's lifetime to a presentation/sheet" | none (§Scenes inline only; no reference file) | none (no scenes.md exists) | **none** |
| R2 | "an @Observable view shows a stale value" | data.md ("stale value / didn't update" bug) | dataflow.md (@Observable per-property tracking, desync) | **ambiguous: data.md vs dataflow.md** |
| R3 | "@State resets when a bool flips" | modifiers.md (conditional modifier causes @State reset) | views.md (.task on conditional branch; @State reset) | **ambiguous: modifiers.md vs views.md** |
| R4 | "is NavigationView deprecated" | navigation.md ("flag all use of deprecated NavigationView") | soft-deprecated-apis.md (NavigationView explicitly listed) | **ambiguous: navigation.md vs soft-deprecated-apis.md** |
| R5 | "app won't quit after last window" (macOS) | none (no scenes.md; §Scenes inline does not cover quit behavior) | none (inline §Scenes covers dismissWindow/onDisappear, not the quit trap) | **none** |

### Summary of failures

- **2 of 5 regression phrases → `none`** (R1, R5): The "scenes" class of questions has no reference file to route to. The inline §Scenes & Windows prose in SKILL.md is never surfaced by reference-based routing.
- **3 of 5 regression phrases → `ambiguous`** (R2, R3, R4): The dual-tree creates multiple equally-plausible landing spots because Apple's set and Paul Hudson's set cover the same domains without a clear priority signal in the descriptions.

---

## Known topic collisions in the current tree

| Topic | Apple file | Hudson file | Collision notes |
|-------|-----------|-------------|-----------------|
| Data flow / @Observable | `dataflow.md` | `data.md` | Both describe stale-value bugs; descriptions nearly identical in intent |
| View structure / composition | `structure.md` | `views.md` | Both cover separate View structs vs computed properties; views.md also covers animation |
| Modern API | `soft-deprecated-apis.md` | `api.md` | api.md repeats a subset of soft-deprecated-apis.md content |
| Navigation / deprecation | `navigation.md` | `soft-deprecated-apis.md` | NavigationView deprecation reachable via both |

---

## Additional routing spot-checks (non-regression)

These phrases were also evaluated to characterize the current routing behavior across the wider topic space.

| Phrase | Expected (current) | Notes |
|--------|-------------------|-------|
| "how do I make a custom animatable shape" | animations.md | Clear, unambiguous |
| "ForEach using array indices as IDs" | foreach.md | Clear, unambiguous |
| "@Entry macro for environment key" | environment.md | Clear; also mentioned in api.md |
| "localize a string in a Swift package" | localization.md | Clear |
| "how to use @State as a cache for CIContext" | data.md or dataflow.md | Ambiguous (dual-tree collision) |
| "conditional modifier breaks animation" | modifiers.md | Clear |
| "LazyVStack for long lists" | performance.md | Clear |
| "@State macro init assignment error" | state-macro.md | Clear (SDK-27 set unambiguous) |
| "ContentBuilder ambiguous use of opacity" | content-builder.md | Clear |
| "APIs hard-deprecated in SDK 27" | deprecations.md | Clear |
