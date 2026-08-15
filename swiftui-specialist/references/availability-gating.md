# Availability Gating — adopting the newest API without dropping the floor

How to ship the API Apple built for a shape when that API is newer than the deployment target.
The rule is in SKILL.md § Primitive-First ("Version floor: hard requirement, not preference");
this file is the mechanics: how to establish availability, the four gating shapes and which one
each situation wants, and how to schedule the fallback's removal so the compiler owns it.

## Contents

- Establishing availability — which tool actually prints `@available`
- Mirror the API's own availability line
- The four gating shapes
- Deprecate your own fallback
- Keep the two paths honest
- What this does *not* license
- Common mistakes

## Establishing availability — which tool actually prints `@available`

| Tool | Prints `@available`? |
|---|---|
| `LSP hover` | **No** — signature + doc comment only |
| `LSP goToDefinition` → generated `.swiftinterface` | **Yes** — attributes are written above the declaration |
| `DocumentationSearch` | No — carries no availability metadata |
| DocC JSON | Yes — `.metadata.platforms` |

`goToDefinition` is the cheap first check and the one to reach for: it lands in the SDK's own
generated interface, where the attribute is verbatim and the neighbouring declarations — overloads,
protocol bounds, `nonisolated` — are visible in the same read. Hover's silence is not evidence a
symbol is unrestricted; hover strips the attribute.

Measured (SDK 27, macOS `26A5399e`): hovering a 27-only `DynamicViewContent` modifier printed the
signature and full doc comment and **no** availability; `goToDefinition` on the same symbol showed
`@available(iOS 27.0, macOS 27.0, watchOS 27.0, *)` immediately followed by
`@available(tvOS, unavailable)`. Two attributes, neither visible to hover.

## Mirror the API's own availability line

Copy the platform list from the declaration just read. Do not derive it, and do not narrow it to
the one platform being built. Per-platform floors differ, and a framework can be *unavailable* on
a platform rather than merely newer there — an annotation that omits that fails the day the target
list grows, at a call site far from the cause. `anyAppleOS N` collapses the matrix where every
platform shares one floor; the explicit list is required where they don't.

Deployment target and SDK are independent axes: building against a newer SDK with an older floor
compiles gated code fine — the gate is what keeps it from *running* there. A symbol missing from
the selected SDK is a different problem; `#available` cannot fix it (shape 3).

## The five gating shapes

Pick by **what differs between the two paths**.

### 1. Branch inside one named `ViewModifier` — same inputs, same shape

The default. Both overloads take the same values and produce the same result; only the spelling
moved. One type, one call site, no `@available` leaking to callers: the `if #available` lives in
the modifier's `body`.

**An `#available` branch cannot flip at runtime.** The OS version is fixed for the process, so the
`_ConditionalContent` it produces never re-roots its subtree — which is why gating inside a
modifier body is safe where a state-driven `if` in the same position is not (`modifiers.md`,
`scenes.md` § Re-rooting, `performance.md`). This is the one conditional that survives the
structural-identity analysis unchanged.

Availability twins are rarely pure renames, and the newer overload is sometimes *less* capable at
the call site — a newer drop-handling overload whose action returns `Void` where the older one
returned `Bool` cannot reject a drop, so a result the old path used gets discarded. Record that
behavioural delta in the doc comment: after gating, no caller can see it.

### 2. Twin types plus one chooser — the signature itself changes

When the new API changes the type's generic requirements or removes a parameter, one body cannot
hold both. Ship two types, annotate each with its own floor, and expose one `@ViewBuilder`
extension that picks between them. Callers then never write `#available`, and the gate exists once.

**Signature parity is the design constraint.** The twin takes the same parameters *minus what the
new API supplies for free*, so moving between them changes one word at the call site and the two
bodies can be read as a diff. Where the new API provides a capability the older one cannot have at
container level, the twin **drops that parameter** rather than faking it, and the caller supplies
the capability per row or per child instead (worked shape: `custom-containers.md` § Reordering).

Both twins need the same builder attribute for parity, and `@ContentBuilder` costs nothing:
measured in SDK 27's interface it is a `typealias` for `ViewBuilder` annotated
`@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *)`. The 27 story is what the builder
*accepts*, not whether the attribute exists (`content-builder.md`), so a pre-27 twin may spell
`@ContentBuilder` too.

### 3. Compile-time `#if` — the symbol is absent from the selected SDK

`#available` is a runtime check on a symbol the compiler can already see. A symbol that is not in
the SDK being built against fails to *compile*, and no runtime gate helps: that needs conditional
compilation keyed on the SDK (`canImport`, or an SDK-scoped
`SWIFT_ACTIVE_COMPILATION_CONDITIONS[sdk=…]` flag).

The two axes **compose**, and usually both are required: the `#if` gets it to compile, the
`#available` nested inside keeps it from running on an older OS. Choosing one axis and declaring
the other impossible is the capitulation to avoid.

### 4. Gate the `#Preview`, and preview both paths

`@available` on a type does not reach its previews — annotate the `#Preview` too.

Preview the fallback **even when the two paths should be identical**. That preview's job is to
prove there is no divergence: where both twins share one mechanism (a scroll-position
implementation whose API predates the split, say) the preview demonstrates the *absence* of a
second mechanism, and it is the only place the gated branch is ever seen. Neither the build nor the
newer-OS preview exercises the `else`.

### 5. Version-free value plus a gated conversion `init` — the newer *type* would go viral

Shapes 1–4 all key on *what differs between the two paths*. This one keys on something else: **`@available` is viral through stored properties.** A newer-OS framework *type* held as a stored member forces that annotation onto your type, then onto every view storing your type, then onto their `@State` / `@Binding` — until the cascade collides with the deployment floor. Branching cannot help, because the question is not which code runs but what can be *stored*.

Declare your own **version-free value** both paths can speak, and put the conversion in an `@available` extension on **your** type:

```swift
// Ungated — stored by views at the deployment floor.
private struct ReorderContext: Equatable {
    enum Destination: Equatable { case before(Item.ID), end, onto(Item.ID) }
    var sources: [Item.ID]
    var destination: Destination
}

@available(iOS 27.0, macOS 27.0, *)
extension ReorderContext {
    /// Gated HERE so no ungated view names a 27-only type. `.onto` is unreachable from a
    /// difference — the exhaustive switch turns a future `Position` case into a compile error
    /// rather than a silent fallthrough.
    init(reorderDifference difference: ReorderDifference<Item.ID, ReorderableSingleCollectionIdentifier>) {
        switch difference.destination.position {
        case .before(let id): self.init(sources: difference.sources, destination: .before(id))
        case .end:            self.init(sources: difference.sources, destination: .end)
        }
    }
}
```

The gate then exists at exactly one point, nothing downstream carries an annotation, and inside an `if #available` block the call site is one line: `request = .init(reorderDifference: difference)`.

**Put it on your type, not as a projection on theirs.** A computed property on the framework type (`var context: ReorderContext`) looks equivalent and is not: that extension is generic over the framework type's *own* parameters, so returning your concrete value needs an extra `where` constraint — which narrows the extension for every other member declared on it, including ones used generically elsewhere. Extending your own type has no such coupling.

**A `where` clause cannot constrain a non-generic type.** `init(_ d: ReorderDifference) where CollectionID == …` fails twice over: the framework type needs its generic arguments spelled, and there is no `CollectionID` in scope to constrain because your type declares no generic parameters. Spell the concrete arguments in the parameter type instead.

The pairing to reach for is `dataflow.md` § "A `Binding` is also an upward intent channel" — a version-free request value is exactly what an ungated container can carry, which is what makes the two patterns compose.

## Deprecate your own fallback

Mark the twin `@available(<platform>, deprecated: <version where the new API landed>, message:)` —
`renamed:` instead when the replacement is a drop-in, which also gets a fix-it.

This compiles clean while the floor is below that version, and the day the deployment target
reaches it every call site warns. The cleanup becomes a compiler task instead of a memory task, and
`message:` names the replacement so the warning is actionable.

Distinct from Apple's soft-deprecation sentinel (`deprecated: 100000.0`, see
`soft-deprecation.md`), which means "discouraged, no version yet." A real version number is a
scheduled removal — that is what a fallback of your own wants.

## Keep the two paths honest

An availability branch duplicates *a spelling*, never *a concern*. When both branches restate the
whole subtree, factor the shared part into one function parameterised by what actually differs
(typically a generic child-builder parameter, so each branch spells the wiring once and the child's
type stays concrete). The moment a fix has to be applied twice, the factoring is wrong — and a
divergence between the paths stays invisible until someone runs the older OS.

## What this does *not* license

- **Not "gate and forget."** A branch with no preview and no test is a branch that has not been
  run. Shape 4 is not optional.
- **Not shipping only the old path** because it also works on the new OS. Same defect as
  hand-rolling a stack where a primitive exists (SKILL.md § Primitive-First).
- **Not shipping only the new path** with a floor raised without permission. The deployment target
  is a product decision; ask.
- **Not a licence to branch inside rows.** Availability branches are process-constant and safe;
  state-driven ones in the same position are not.

## Common mistakes

| Mistake | Reality |
|---|---|
| "Hover showed no `@available`, so it's unrestricted" | Hover strips the attribute. `goToDefinition` → generated `.swiftinterface` prints it |
| Annotating a wrapper with one platform's floor when the API names several plus an *unavailable* | Copy the API's own line. A derived floor breaks on the platform you didn't build |
| `if #available` for a symbol the selected SDK lacks | Runtime gate, compile-time problem. Needs `#if` — and then usually both |
| Twin types with divergent parameter lists | Parity is the point: same call site minus what the new API supplies. Diverge, and neither reader nor caller can compare them |
| Fallback twin left un-deprecated | `deprecated:` at the version the new API landed makes the compiler schedule the cleanup |
| `deprecated: 100000.0` on a type of your own | That sentinel is Apple's "discouraged, no version." Yours has a version — use it |
| Both availability branches restating the whole subtree | Duplicate the spelling, not the concern. Parameterise one function by what differs |
| No preview for the `else` branch | Nothing else exercises it. Preview it even when it "should be identical" — that is what the preview proves |
| Treating an `#available` branch as a structural-identity hazard | It cannot flip at runtime. The hazard is the state-driven branch, not this one |
| Storing a newer-OS framework type on a type your ungated views hold | `@available` is viral through stored properties — your type, its views, their `@State`/`@Binding`, all annotated, then the floor breaks. Shape 5: version-free value + gated conversion `init` |
| `where CollectionID == …` on an `init` in an extension of a non-generic type | Nothing to constrain — your type declares no generic parameters, and the framework type still needs its arguments spelled. Concrete parameter type, no `where` |
