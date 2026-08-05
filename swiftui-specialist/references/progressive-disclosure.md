# Progressive Disclosure — designing the API you're about to write

How to shape a reusable component's own API: what belongs in the simple call site, what becomes
an overload, and when to stop adding parameters and start composing. Apply whenever you write a
type or modifier someone else (including future you) will call.

Source: WWDC22, *The craft of SwiftUI API design: Progressive disclosure* (SwiftUI team).
The principles are current; some API spellings in that talk are 2022-era — `Layout` postdates
it, and `.navigationBarLeading` has since been superseded. Take the reasoning, verify the symbol.

## Contents

- The definition, and the perspective it depends on
- Four strategies
- The two questions that decide every overload
- Worked example: `Table`'s simplification chain
- Compose, don't enumerate
- What this does *not* license
- Common mistakes

## The definition, and the perspective it depends on

> **Progressive disclosure is designing APIs so that the complexity of the call site grows with
> the complexity of the use case.**

An ideal API is simple and approachable *and* accommodates powerful use cases. Not a trade-off
between the two — the powerful case pays for its own complexity, and the simple case doesn't.

The whole principle rests on one perspective shift. You write code at the **declaration site**,
so that is where you naturally judge it — a stored property, one more init parameter, a generic
placeholder all look cheap there. The only place that matters is the **call site**, where the
cost is paid on every use. Judge from there.

The macOS save dialog is the UI form of the same idea: a default location already filled in, a
drop-down of likely locations, and a full filesystem browser one disclosure away. Layers of
complexity, revealed on demand.

Why it's worth the effort: it minimises time-to-first-build-and-run, keeps the learning curve
flat by not forcing irrelevant concepts on simple cases, and produces a tight feedback loop —
you add a piece at a time and see the result at each step. Building becomes a cycle of quick
refinements instead of one large up-front investment.

## Four strategies

1. **Consider common use cases.** Identify what the simple case *is* before designing for it.
   Apple's canonical instance is the label pattern — `Button("Next Page") { … }` for the ~99%
   case, plus a `label:` overload taking an arbitrary view for the rest. That pair repeats across
   the entire framework.
2. **Provide intelligent defaults.** Everything the simple case doesn't specify must still do
   the right thing. `Text` localises through the bundle, adapts to colour scheme, scales with
   Dynamic Type, and gets correct line spacing between stacked siblings — all overridable, none
   of it at the call site. `.toolbar` places items by platform convention, with explicit
   placement available when needed.
3. **Optimise the call site.** Every character should serve a purpose. This is where conveniences
   get added — see the `Table` chain below.
4. **Compose, don't enumerate.** Covered in its own section; it's the one that changes the
   *shape* of the API rather than trimming it.

## The two questions that decide every overload

Ask both at every step:

- **What are the most common use cases we should build conveniences for?**
- **What is the essential information that should always be required?**

The second question is the one that keeps the first honest. A convenience is only a convenience
if what it omits was genuinely non-essential; omitting required information produces an API that
compiles and misbehaves.

Applied to a container: if the caller must hand you content your type doesn't need in order to do
its job, that content is not essential information, and requiring it is a call-site tax on every
use. (See `custom-containers.md` § Step 0 — the same question decides whether a selector-style
container should render the selected pane at all.)

## Worked example: `Table`'s simplification chain

Four successive optimisations, each justified by one of the two questions. Worth studying because
each step is a shape you will reach for yourself.

**1. Fully specified** — an explicit `ForEach` in `rows:`, a view closure per column, a
`sortOrder` binding:

```swift
Table(sortOrder: $sortOrder) {
    TableColumn("Title", value: \Book.title) { book in Text(book.title) }
    TableColumn("Author", value: \Book.author) { book in Text(book.author) }
} rows: {
    ForEach(currentlyReading) { book in TableRow(book) }
}
```

**2. Pass the collection directly.** Most `rows:` blocks are exactly a `ForEach` over a
collection, so `Table` supplies that `ForEach` behind the scenes. This is the data-driven
convenience — see `custom-containers.md` § L1, where it is the layer's precedent:

```swift
Table(currentlyReading, sortOrder: $sortOrder) {
    TableColumn("Title", value: \.title) { book in Text(book.title) }
    TableColumn("Author", value: \.author) { book in Text(book.author) }
}
```

**3. String key paths omit the column view.** When `value:` points at a `String`, the closure is
exactly `Text(book.title)` — so it may be dropped entirely:

```swift
Table(currentlyReading, sortOrder: $sortOrder) {
    TableColumn("Title", value: \.title)
    TableColumn("Author", value: \.author)
}
```

**4. Drop the concern the simple case doesn't have.** The simplest table doesn't sort, so there is
an overload with no `sortOrder` at all — and the `.onChange` re-sort that came with it disappears
from the call site too:

```swift
Table(currentlyReading) {
    TableColumn("Title", value: \.title)
    TableColumn("Author", value: \.author)
}
```

Step 4 is the one most often skipped. Sorting isn't given a default; the parameter is *absent*
from that overload. A concern the simple case does not have should not appear in its signature
with a placeholder value — the presence of the parameter is itself the complexity.

**All four coexist — the chain is about call sites, not about which initialisers exist.** This is
the step that is easiest to misread. `Table` did not migrate from 1 to 4 and delete the general
form; it *added* conveniences beside it. Verified in SDK 27: `init(sortOrder:columns:rows:)` and
`init(_:sortOrder:columns:)` both ship, across roughly twenty initialisers spanning both families.
Progressive disclosure is layers existing simultaneously — so this chain is never an argument for
deleting a builder init. If you find yourself citing it that way, you have inverted it.

Note also what the convenience costs Apple: `init(_:sortOrder:columns:)` is constrained
`where Rows == TableForEachContent<Data>` — it must name the concrete type its delegation
produces. That is the same pin a custom container pays as
`Content == ForEach<Data, Data.Element.ID, RowContent>` (`custom-containers.md` § L1). Apple pays
it deliberately, to keep both forms. The pin is the price of coexistence, not evidence of a
design smell.

## Compose, don't enumerate

The failure mode: designing an `arrangement` enum for `HStack` with `.leading`, `.center`,
`.trailing`. It covers the cases named — then someone wants elements spread evenly, or space only
between them, or space only before the last one. Each new behaviour needs a new case, the set is
unbounded, and you will not think of them all up front.

SwiftUI ships `Spacer` instead. One composable piece, placed by the caller, expresses every
arrangement above and many that were never enumerated:

```swift
HStack { a; b; c }                             // leading
HStack { Spacer(); a; b; c; Spacer() }         // centred
HStack { Spacer(); a; Spacer(); b; Spacer(); c; Spacer() }   // evenly spread
HStack { a; Spacer(); b; Spacer(); c }         // space between only
HStack { a; Spacer(); b; c }                   // space after the first only
```

Five arrangements from one primitive, and the last two were never in the enum.

**The tell: you are adding a case per behaviour to an enum, a parameter per behaviour to an init,
or a boolean per behaviour to a config struct.** Stop and look for the one composable piece the
caller can position themselves.

This is not merely about a shorter call site — it's about how the call site *scales*. An
enumerated API's simple case can look perfect and still be the wrong design, because the
complexity it can express is capped at what you predicted.

## What this does *not* license

- **Not "fewer parameters is better."** The goal is that complexity *tracks* the use case.
  Removing essential information to shorten a signature produces an API that can't express its
  own job.
- **Not "add a default to everything."** Step 4 above removes the parameter for the case that
  doesn't need it. A default value still puts the concept in the signature, where every reader
  pays to understand it.
- **Not a licence to hand-roll.** Progressive disclosure governs the API of a component you have
  already established needs to exist. Whether it should exist is the primitive-first question —
  see SKILL.md § Primitive-First.
- **Not an environment-modifier rule.** Moving a knob to an `@Entry` plus a `View` extension is
  one mechanism serving strategy 2, useful when a value must skip intermediate views. The
  principle's primary mechanisms are overloads, defaults and composition. Reaching for the
  environment first inverts that — and closures must never go there at all
  (`environment.md` § Closures in the Environment).

## Common mistakes

| Mistake | Reality |
|---|---|
| Judging the API from the declaration site | That's where you write it, not where it's paid for. One more parameter is cheap to declare and costly at every call |
| Giving the simple case a parameter with a default | `Table` *omits* `sortOrder` for the non-sorting case. A default still puts the concept in the signature |
| An enum case (or init parameter, or bool) per behaviour | You cannot enumerate what callers will want. Ship the composable piece — `Spacer`, not an `arrangement` enum |
| Keeping a builder init no caller uses | Dead API is worth removing on its own merits — but the `Table` chain is not the argument for it. `Table` kept `rows:` and added the convenience beside it |
| Citing the `Table` chain to justify deleting a general form | Backwards. The chain *added* overloads; all of them still ship. Progressive disclosure is layers coexisting |
| "Simple *or* powerful — pick one" | The premise is wrong. The powerful case pays its own complexity; the simple case shouldn't subsidise it |
| Treating a shorter call site as the goal | The goal is that it *scales*. An enumerated API with a beautiful simple case is still capped at what you predicted |
