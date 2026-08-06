# Custom Containers

How to build a container view — a type that takes caller-supplied content and arranges,
selects, or decorates it. Covers the API contract, which Apple layer to reach for, and the
research method that lands you on the right one.

## Contents

- The method — three steps, in order
- Step 2's failure mode: semantic overlap is not structural precedent
- Step 0: decide what the container owns
- The layer ladder — L0 to L5
- Which tool answers which question
- Facts per layer
- Reordering
- Refactoring an existing container
- Common mistakes

## The method — three steps, in order

1. **Break the view into separate `View` types.** Sections become types, not `private var`
   computed properties or `@ViewBuilder` methods. See `structure.md`.
2. **Find precedent in Apple's own APIs** — the shipped container whose *structure* matches
   what you're building.
3. **Read the entire doc for that precedent, not just its sample code.** The Overview and
   Discussion state purpose and applicability; the declaration states only what compiles.
   Applicability is what decides which layer you're on.

Step 3 is the one that gets skipped. A signature tells you a type exists and what it
accepts. Only the Overview tells you what it is *for* — and a container built from the right
signature at the wrong layer compiles, renders, and is still wrong.

> "Break down into separate subviews" (step 1) means separate `View` **types**. It is
> unrelated to `Group(subviews:)`, which decomposes an existing view. Do not conflate them.

## Step 2's failure mode: semantic overlap is not structural precedent

Pick the precedent whose **structure** matches, not whose **subject matter** matches.

Worked example. Building a custom tab bar to replace `TabView` (because `TabView` can't
reach the required feature set):

| Candidate | Subject match | Structural match | Usable? |
|---|---|---|---|
| `TabView` / `TabContent` / `TabContentBuilder` | exact — it is literally tabs | no — `TabContent` is the private contract `TabView` consumes | **No.** Conforming to it makes your type `TabView` content, which is the thing you're replacing |
| `List(_:selection:rowContent:)` | none — it's a list | yes — container + caller-supplied row content + selection | **Yes** |

The subject-matching API was the wrong answer and the subject-distant one was right. When a
framework type shares your domain vocabulary, that is *weak* evidence — check whether its
contract is one you can actually join, or whether it's internal plumbing for the very
component you're replacing.

## Step 0: decide what the container owns

The ladder answers *how* you gather content. Answer *what you own* first — it sets the generic
signature, so changing your mind later is a rewrite, not a refactor.

| Apple's | Owns | Caller owns |
|---|---|---|
| `Picker`, `Slider`, `Stepper` | the selector, nothing else | placement of whatever the selection drives |
| `List`, `Table` | rows | the surrounding scene |
| `TabView`, `NavigationSplitView` | selector **and** selected content **and** the hosting | almost nothing |

Own both halves only if you also own the host. `TabView` may place its own bar because it also
decides the chrome it sits in. (`TabView` is the exhibit for the ownership *bargain* here — it is
still not a contract to join or a signature to model; see Step 2.) A container that renders the
selected pane *and* picks its own host — `.toolbar` here, a safe-area bar there — has taken the
caller's layout decision without taking `TabView`'s responsibility for it.

**Default to the selector alone.** A selector-only container composes into whatever host the
caller already has, and drops a whole generic parameter: the `Content` type that existed only to
render the pane is dead weight at every call site that wanted just the bar. Owning the pane also
pulls in the selected view's lifetime — remount-on-change, `.id(selection)`, task cancellation —
which is the caller's concern in every shape except `TabView`'s.

## The layer ladder — L0 to L5

Descend only when the layer above genuinely cannot express the shape. Each descent has a
stated price; skipping down is how imperative habits re-enter declarative code.

| | Layer | Reach for it when | Apple's |
|---|---|---|---|
| **L0** | `Group { }` — collect, never decompose | fan one modifier over several views · one modifier across an `if`/`else`'s branches · relieve builder child-count | `Group` |
| **L1** | Data-driven init pinning `Content == ForEach<Data, Data.Element.ID, RowContent>` | you hold the model collection | `List(_:selection:rowContent:)` |
| **L1b** | Builder init bounded `Content: DynamicViewContent` — caller supplies the `ForEach`, container *reads* `content.data` | you need the collection (count, identity type) but not the row type, and row concerns can ship as caller-applied modifiers | every `DynamicViewContent` modifier: `onDelete`, `onMove`, `reorderable` |
| **L2** | Typed element + `@resultBuilder` over **values** | children are hand-authored/heterogeneous, or one element must carry more than one view plus a value | `Tab(value:content:label:)` + `TabContentBuilder` |
| **L3** | Named `ViewModifier` per row concern | the container wraps rows *and* pins `Content` | `ModifiedContent` |
| **L4** | `Group(subviews:)` / `ForEach(subviews:)` / `ContainerValues` | the only input is an opaque builder block, and placement is positional or driven by child-written values | `Group(subviews:transform:)` |
| **L5** | `Layout` | geometry across siblings; distributing an offered width | `Layout` protocol |

`Group { }` (L0) and `Group(subviews:)` (L4) are **different initializers with opposite
jobs** — one collects, one decomposes. Apple's own words for L0: "collect multiple views
into a single instance, *without affecting the layout of those views*, like an `HStack`,
`VStack`, or `Section` would."

That layout-neutrality is what makes the ladder progressive: because collecting imposes no
geometry, you can swap the layout (L5) without touching how content was gathered. `HStack`,
`VStack` and `Section` all fail that test — they impose geometry, and `Section` also imposes
platform grouping and hierarchy.

**The dominant error is descending too far, not too little.** If you hold the model
collection, you are at L1 and no decomposition is needed — the container renders the
`ForEach` and never inspects it. Reaching L4 to recover a value the model already holds
inverts the data flow: the model is the source of truth, never the view tree.

## Which tool answers which question

| Question | Tool |
|---|---|
| What is this type *for*? When does it apply? | `DocumentationSearch` — returns Overview and Discussion |
| How is it actually implemented / declared? | `AppleSourceCodeSearch` (filters are separate params, not query text) |
| What OS introduced it? Is it deprecated? | DocC JSON: `curl …/documentation/<fw>/<symbol>.json \| jq '.metadata.platforms, .deprecationSummary'` |

**The DocC JSON endpoint returns `.abstract` — one line — and its
`.primaryContentSections[…kind=="content"]` is frequently empty.** It cannot answer "what is
this for." Reading a one-line abstract and proceeding as though the doc was read is a
measured failure mode: it is how `Group`'s three documented purposes get missed entirely.
Use it for availability, never for purpose.

`LSP hover` strips `@available` and `DocumentationSearch` carries no availability metadata — so
neither tool's silence is evidence a symbol is unrestricted. `LSP goToDefinition` does print it:
it lands in the generated `.swiftinterface`, where the attributes sit above the declaration.
Adopting a version-gated container primitive: `availability-gating.md`.

## Facts per layer

**L1 — two shapes, pick by whether you also have a builder init.**

*Data-driven only* — store the collection and a per-element row builder; construct the
`ForEach` in `body`. No `Content ==` constraint, no nameable-wrapper requirement. This is the
default and the simpler shape:

```swift
struct FilterBar<Data: RandomAccessCollection, ID: Hashable, Content: View>: View {
    let data: Data
    let id: KeyPath<Data.Element, ID>
    @Binding var selection: ID?
    let content: (Data.Element) -> Content        // per-element factory, like ForEach's own
}
```

*Builder init plus a data-driven convenience* — the convenience delegates to the builder, so
`Content` must name the concrete result: `Content == ForEach<Data, Data.Element.ID,
RowContent>`. This is `List`'s shape, and it is what drags in L3's nameable-wrapper rule.
Only take it on if callers genuinely need the builder form.

**The pin is the price of shipping both, and Apple pays it too.** `Table`'s data-driven
convenience is constrained `where Rows == TableForEachContent<Data>` for exactly this reason,
while `init(sortOrder:columns:rows:)` continues to ship alongside it. Verified in SDK 27. So the
`Content ==` constraint is not a reason to drop the builder init — it is what having both costs.
Decide the builder's fate on whether callers need it (heterogeneous or hand-authored children, or
one element carrying more than one view plus a value ⇒ keep it, and see L2), never on the pin
being ugly. See `progressive-disclosure.md` § Worked example for the full chain and the
misreading to avoid.

Either way `Data` is the **model** collection, `Data.Element: Identifiable` is a model
element, and `ForEach` owns identity, laziness and diffing. Add
`Data.Element.ID == SelectionValue` when the container selects, mirroring `List`'s
selection-aware inits.

**L1b — bound `Content: DynamicViewContent` when the container needs the *data*, not the rows.**

`DynamicViewContent` refines `View` with a single requirement: `var data: Self.Data { get }`, where
`Data` is a `Collection`. Bounding a builder init's result to that protocol — plus
`Content.Data.Element: Identifiable`, and `Content.Data.Element.ID == SelectionValue` where the
container selects — buys read access to the caller's collection while the caller keeps the
`ForEach`. Element count for width distribution, the identity type for selection, and the whole
`DynamicViewContent` modifier family become reachable without pinning `Content`.

Consequences, in the order they bite:

- **`data` is get-only, so the container can never be the mutation site.** Reorder, delete and
  insert must be handed *out* — as a difference, an `IndexSet`, an offset — to whatever owns the
  collection. A container that wants to mutate the caller's array is at the wrong layer, or wants
  L1's data-driven init instead.
- **No `Content ==` pin, so L3's nameable-wrapper cascade never starts.** This is the layer to try
  when a builder init is genuinely wanted *and* the row-wrapper types are turning into a
  combinatorial signature. L5 is unaffected: the container still owns its `Layout`.
- **Row concerns become caller-applied modifiers**, because the container holds pre-built content
  and cannot wrap rows without renaming their type. Ship them as named `ViewModifier`s named for
  the *concern* — layout, choice, drag — one per concern.
- **Apply order is part of the contract, not a style preference.** A hit-test shape has to land
  inside the `Button` a selection modifier introduces; a drop target has to cover the widened frame
  rather than the label. Both orderings fail silently when wrong, so state the order on the
  modifier that must come second, naming the one it follows.
- **A call site can omit a modifier and get an inert row, with no diagnostic.** A row carrying only
  a label compiles, lays out, and does nothing: no selection, no width floor. That is this layer's
  real price, and it is paid at every call site instead of once in the type. Mitigate with a
  `#Preview` of the *omitted-modifier* call site — so the failure has a reference rendering — and by
  keeping the modifier set small enough to state in one line.

Choosing between them: **L1** when the container should own the row's whole appearance and callers
never compose per-row behaviour; **L1b** when callers must attach behaviour the container cannot
generalise (a per-row drag source below a version floor, a per-row context menu, per-row gestures),
or when two different row types must coexist in one container — which L1's single `RowContent`
cannot express.

**L1/L3 — `.tag()` writes an association nothing public can read.** `Picker` and `TabView`
read tags through private machinery. A custom container must wire selection explicitly — the
element's value plus the binding, held by a modifier. A tag you write and never read is
decoration.

**The AX representation is a fake, and must not share the real binding.** A custom selection
control reports the wrong role — a bare `Button` is `AXButton`, `Toggle` is `AXCheckBox` — so the
role has to come from `accessibilityRepresentation` substituting a stand-in. That stand-in exists
only to carry the role and the selected state, and it is built **per item**:

```swift
// One tag, and a constant that matches it iff this item is selected.
.accessibilityRepresentation {
    Picker(selection: .constant(isSelected ? 0 : 1)) {
        label.tag(0)
    } label: { label }
    .pickerStyle(.radioGroup)
}
```

Two failure modes, both of which compile clean:

- **Passing the real selection binding.** `Picker(selection: $selection) { label.tag(0) }`
  requires every tag to be the selection's own type. A literal `0` against a generic `Value?`
  matches only when `Value == Int` *and* the value is `0`; for every other element, and for every
  other `Value`, the representation reports nothing selected. Tag matching is `AnyHashable`-based
  at runtime, so there is no diagnostic and no crash — just a silently wrong AX tree.
- **Expecting the stand-in to model the whole group.** Each item substitutes its own
  single-option picker; the role comes from the substitution, not from a real radio group. So the
  stand-in needs exactly one tag plus a constant that either matches it or doesn't — a second
  option, or a shared binding, buys nothing.

This is the AX twin of the `.tag()` rule above. Real selection is wired through the element's
value and the binding; the AX stand-in is wired through a constant derived from that same
comparison. Neither one reads a tag the framework wrote.

**L3 — every row wrapper must be a *named* `ViewModifier`.** `.frame`, `.contentShape` and
most modifiers return opaque `some View`, which cannot be spelled in a `where` clause. If the
init wraps rows and pins `Content`, the row type must be nameable:

```swift
Content == ForEach<Data, Data.Element.ID,
    ModifiedContent<ModifiedContent<RowContent, ItemLayout>, ItemChoice<SelectionValue>>>
```

Constraint, not inconvenience — and it forces one named type per concern, which is the right
factoring anyway. Named wrappers cannot be `private`: they appear in the public signature.

**L3 and L5 on the same container is a signal, not a configuration.** L3 exists because a builder
init forced `Content` to be nameable; L5 exists because siblings must share an offered width.
Needing both means you are paying for the builder init in row-wrapper types *and* still not
getting the geometry from them — drop the builder, fall back to L1, and the named wrappers
collapse with it. The other exit is L1b: keep the builder, bind `Content: DynamicViewContent`, and
the container still reads the element count its `Layout` needs while the wrappers move to the call
site and the pin disappears. Re-run this check after each concern lands, not once at the start: the
layer was chosen before you knew the full concern list.

Watch for the inverse tell. Once row concerns are factored into per-row modifiers, every row
sizes itself and a plain `HStack`/`LazyHStack` starts to look sufficient — so the *absence* of
width distribution stops being visible at the call site. A container that has row wrappers but no
`Layout` is the shape to re-examine.

**Selection: `Binding<V?>` vs `Binding<V>?`** are different axes. An optional *binding* means
selection is opt-in (`List` — most lists have none). A binding to an *optional* means absence
is representable. A container that is meaningless without selection wants the latter.

**L5 — a `Layout` cannot read `@Environment`.** `sizeThatFits`/`placeSubviews` are
nonisolated protocol methods with no environment access; per-subview data travels via
`LayoutValueKey`. The view *constructing* the layout reads the environment and passes a plain
value.

**Sizing defaults go in the environment only when they must skip intermediate views.** If the
container builds the consumer itself, one level down, pass a plain value — a single
producer→consumer edge earns no indirection. Reach for the environment when the value would
otherwise be threaded as a parameter through views that don't use it. Precedent:
`EnvironmentValues.defaultMinListRowHeight` (macOS 10.15). Use `@Entry` with a **literal**
default — a computed default re-evaluates on every fallback read and invalidates readers on
unrelated environment writes.

**Nothing fills a scroll container's scroll axis.** A horizontal `ScrollView` proposes
unbounded width, so `.frame(maxWidth: .infinity)` on children is inert — measured: five
120pt-floored items in a 520pt bar overflow rather than compressing to 104pt each. Inject a
finite proposal (`containerRelativeFrame`) and handle the nil/infinite case explicitly in the
`Layout`:

```swift
// `nil` = "what do you want?"; `.infinity` = "as much as you like" — neither is a width to fill.
guard let offered = proposal.width, offered.isFinite else {
    return CGSize(width: count * preferredWidth, height: height)
}
```

The injected proposal is where the element count earns its keep: proposing
`max(offered, count × floor)` — one `containerRelativeFrame` closure reads both — makes the row fill
the container while there is room and report *true overflow* past the floor, which is the only thing
an enclosing `ScrollView` can scroll to. **Placed width must equal reported width.** Clamping
children up to a floor while reporting only what was offered puts the tail outside the bounds:
drawn, unreachable, and observable as rubber-banding rather than scrolling.

This is a *proposal* property, not a laziness property — it applies to `HStack` and
`LazyHStack` alike. Laziness costs something different and independent: off-screen children
are neither accessibility elements nor drop targets.

## Reordering

Drag-to-reorder is the concern that most often reveals the container is at the wrong layer, because
the mechanism differs by version *and* by whether the container can reach its rows.

**`onMove(perform:)` installs nothing outside a `List`.** It is declared on `DynamicViewContent`
(macOS 10.15+), so it compiles on a `ForEach` inside a stack, a grid, or a custom layout — and does
nothing there. No lift, no drag, no diagnostic. It writes the `List` reorder trait, and nothing else
consumes it; Apple scopes it twice, in *Making a view into a drag source* ("**Within a `List`**, you
can use the `onMove(perform:)` modifier to enable reordering") and in `EnvironmentValues.editMode`,
which describes the affordance as belonging to "a `List` with a `ForEach`". A compiling `onMove` on
anything else is the single most convincing dead end in this area.

**The container-level mechanism (27+)** is a pair: `reorderable()` on the `DynamicViewContent`, and
`reorderContainer(for:isEnabled:move:)` on the enclosing stack, grid, or custom layout. The item
type must be `Identifiable` with a `Sendable` ID. The `move` closure receives a difference carrying
the source item IDs, in the order the person selected them, plus a destination position expressed as
*before a given item ID* or *at the end*.

Applying that difference:

- **The destination is in the original index space**, so a straight application needs no off-by-one
  correction. `Array.move(fromOffsets:toOffset:)` is the opposite: it uses the *pre-removal*
  convention and needs `+1` when moving an element later in the collection. Two conventions, two
  translations, and a wrong offset is wrong in one direction only — so test both directions of every
  translation you write, never just one.
- **Apply by ID, not by index.** Keying the collection into an ordered dictionary by element ID turns
  the whole difference into one move-keys call and writes back in order; index arithmetic over a
  multi-source difference is where the fudge factors come from.
- **The container is not the mutation site** when content is bound rather than pinned (L1b): pass the
  difference out to the collection's owner.

**Below the container-level floor there is no equivalent**, so the fallback is per-row: a drag source
plus a drop target on each row. A container holding pre-built content cannot attach those, so they
become a caller-applied row modifier — which is exactly why the pre-floor twin *drops* the reorder
parameter instead of faking it (`availability-gating.md` § shape 2). Per-row translation is a
hit-test ("dropped item onto this row"), not an index difference; it needs its own test.

Measured hazards, all silent (macOS 27, `26A5399e`):

- **A state-driven `if` in a row's subtree breaks the lift.** Once the reorderable modifier is
  installed, a `_ConditionalContent` in the row defeats the drag gesture. Every row-level toggle —
  selection chrome, badges, an underline — becomes an `opacity` ternary. Run the structural-identity
  analysis on the row before adding any conditional (`scenes.md`, `performance.md`).
- **A glass effect anywhere in a reorderable item's ancestor chain renders the drag preview empty.**
  Reproduced in Apple's own documented shape, so it is not fixable by re-composing the call site; use
  a material until the platform fixes it.
- **The default drag preview is the row's snapshot**, so a row that fills the container's width drags
  at full width. Supply an explicit preview for anything wider than its content.
- **A reorder can re-anchor the scroll view mid-drag.** Pin the scroll anchor to the dragged item for
  the drag's duration (drag-session updates, 26+, itself gated).

## Refactoring an existing container

Moving a container to a different layer — or rewriting it beside the old type — re-derives every
sub-concern the old one already solved. Inventory them before writing anything: the row view, the
`Layout`, the AX representation, the drag plumbing. Reuse those types. Re-implementing one from
its doc comment is how a working behaviour silently stops working, because the comment records
*what* must hold and rarely *why that construct was the one that held it*.

**A preserved invariant comment is not evidence the invariant survived.** The comment travels with
a refactor — it reads as documentation, so it gets carried onto the new type. The mechanism does
not travel, because reproducing it requires knowing what was measured. The result is a comment
stating the correct requirement sitting directly above code that violates it, which reads as
*verified* to every later reader, including the person who wrote it.

Measured: `INVARIANT: the AX representation must yield AXRadioButton — five XCUITests assert it`
was carried verbatim onto a new row modifier whose `Picker` bound the real selection against a
literal tag, so it matched nothing (see the AX-representation fact above). Comment right, code
wrong, pair indistinguishable from correct.

So, when a doc comment states a measured invariant:

- Re-verify the **mechanism** against the original implementation, never against the comment.
- If you cannot say why the original picked that construct, you are not yet in a position to
  replace it — go read what it was measured against (the test, the probe, the ADR it cites).
- Prefer moving the original type over re-deriving an equivalent. A type that already passes its
  tests carries the invariant for free.

This applies with full force to your own earlier work. Authorship is not understanding; the
refactor that loses the invariant is usually done by someone who has read the comment many times.

## Common mistakes

| Mistake | Reality |
|---|---|
| Reading the declaration and skipping the Overview | Signatures say what compiles; Overviews say what applies. The layer choice comes from applicability |
| Picking the precedent that shares your domain words | Match structure, not subject. The tabs-named API may be internal plumbing for the component you're replacing |
| "No public API exists for this" | Check the current SDK before repeating it — and check whether a *shallower* layer removes the need entirely |
| Relocating a `for` loop into an `init` | Same eager construction behind a nicer signature. Not a fix |
| Reaching L4 to recover an element's value | The model already holds it. Reading views to recover model data inverts the flow |
| `Group { SingleConcreteView() }` | Adds a type wrapper every chained modifier type-checks against. `Group` around an `if`/`else` is *not* this anti-pattern — its content is `_ConditionalContent` |
| Treating `Section.header`/`.content` as two slots to re-place | `Section` is a semantic container, not a neutral carrier. Fights the type's purpose and the data flow at once |
| Rendering the selected content pane *and* picking your own host | That is `TabView`'s bargain, and it only works because `TabView` owns the chrome too. Ship the selector; let the caller place the pane |
| Carrying an `INVARIANT:` comment onto a rewritten type | The comment travels, the mechanism doesn't. Comment-right/code-wrong reads as verified. Re-verify against the original implementation |
| Row wrappers (L3) but no `Layout` (L5) | Self-sizing rows hide the missing width distribution at the call site. Needing both layers means the builder init isn't paying for itself |
| `accessibilityRepresentation { Picker(selection: $selection) { label.tag(0) } }` | The stand-in must not share the real binding. Per-item, one tag, `.constant(isSelected ? …)` — a literal tag against a generic `Value?` matches nothing and never diagnoses |
| `onMove(perform:)` on a `ForEach` outside a `List` | Compiles, installs nothing, warns about nothing. Container-level reorder is the 27+ `reorderable()` + `reorderContainer(for:)` pair; below that floor it is per-row drag + drop |
| Expecting a container bounded to `DynamicViewContent` to mutate the collection | `data` is get-only. Hand the difference, `IndexSet` or offset out to whatever owns the collection |
| Shipping caller-applied row modifiers with no omitted-modifier preview | An inert row compiles and renders. That preview is the only reference for the failure mode the layer introduces |
| A state-driven conditional inside a reorderable row | `_ConditionalContent` defeats the drag lift. Ternary the value; never branch the view |
| Reporting the offered width while placing children wider | Placed must equal reported. Otherwise the tail is drawn outside the bounds, unreachable, and reads as rubber-banding |
