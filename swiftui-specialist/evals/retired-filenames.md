---
record: retired-filenames-inventory
version: "1.0"
created: "2026-06-24"
task: "Task 2 — inventory consumers of retired filenames"
status: record-only
note: "Consumed by the deferred enforce-on-read hook effort; no task in THIS plan consumes it directly. Every bucket-A consumer has a fix mapped to Tasks 4–6."
---

# Retired-filename inventory

This file is **record-only**. It is the authoritative reference for the deferred
`enforce-on-read` hook effort (post-Tasks 4–6), which will verify that no loaded skill
ever routes a reader to a retired filename. No task in the current restructure plan
consumes this file directly; the fixes below are applied in Tasks 4, 5, and 6.

---

## Retired set (6)

The following 6 reference files are being deleted as part of the restructure:

| Filename | Path in `swiftui-specialist/references/` | Is symlink? | Symlink target |
|---|---|---|---|
| `views.md` | `references/views.md` | yes | `../../swiftui-pro/references/views.md` |
| `data.md` | `references/data.md` | yes | `../../swiftui-pro/references/data.md` |
| `hygiene.md` | `references/hygiene.md` | yes | `../../swiftui-pro/references/hygiene.md` |
| `navigation.md` | `references/navigation.md` | yes | `../../swiftui-pro/references/navigation.md` |
| `design.md` | `references/design.md` | yes | `../../swiftui-pro/references/design.md` |
| `api.md` | `references/api.md` | yes | `../../swiftui-pro/references/api.md` |

All 6 are symlinks into `third-party/swiftui-pro/swiftui-pro/references/`.
Deleting them in Tasks 4/5 removes only the symlink — the real upstream file is untouched.

---

## Surviving post-restructure set (16)

The canonical filename set after the restructure. The enforce-on-read hook should
accept these as valid targets and reject any of the 6 retired names above.

```
structure.md
dataflow.md
environment.md
foreach.md
modifiers.md
animations.md
localization.md
soft-deprecation.md
soft-deprecated-apis.md
performance.md
swift.md
accessibility.md
scenes.md          (new — created in Task 3)
state-macro.md
content-builder.md
deprecations.md
```

---

## Bucket A — live consumers (WILL BREAK on retirement)

These are hardcoded references inside the `swiftui-specialist` skill itself. Each one
will dangle after the corresponding file is deleted. A planned fix and the task that
applies it are recorded for each.

| file:line | exact text | planned fix | task |
|---|---|---|---|
| `swiftui-specialist/SKILL.md:229` | `` - `references/api.md` — modern API and the deprecated code it replaces. `` | Remove this `## References` bullet (api.md content is already fully covered by inline §API + `soft-deprecated-apis.md`) | Task 5 |
| `swiftui-specialist/SKILL.md:230` | `` - `references/views.md` — view composition and animation. `` | Remove this bullet; nuggets migrated to `structure.md` / `animations.md` / `scenes.md` | Task 4 |
| `swiftui-specialist/SKILL.md:231` | `` - `references/data.md` — data flow, shared state, property wrappers (overview). `` | Remove this bullet; nuggets migrated to `dataflow.md` / `scenes.md` | Task 4 |
| `swiftui-specialist/SKILL.md:232` | `` - `references/navigation.md` — `NavigationStack`/`NavigationSplitView`, alerts, dialogs, sheets. `` | Remove this bullet; Liquid-Glass `confirmationDialog` rationale migrated to inline §Navigation | Task 5 |
| `swiftui-specialist/SKILL.md:233` | `` - `references/design.md` — accessible apps meeting Apple's HIG. `` | Remove this bullet; `LabeledContent`-outside-Form + custom style migrated to inline §Design | Task 5 |
| `swiftui-specialist/SKILL.md:237` | `` - `references/hygiene.md` — clean compilation and long-term maintainability. `` | Remove this bullet; hygiene.md is lossless (all content already inline §Hygiene) | Task 5 |

After Tasks 4 and 5 have applied these per-bullet removals, Task 6 rebuilds the
`## References` list in full to reflect only surviving files and the new `scenes.md`.

### Out-of-scope hits (swiftui-pro upstream SKILL.md)

`rg` also found matching lines in:
- `third-party/swiftui-pro/swiftui-pro/skills/swiftui-pro/SKILL.md` (lines 15–19, 23, 102–109)

These reference `${CLAUDE_SKILL_DIR}/references/api.md` etc. inside the separate
`swiftui-pro` skill (a different skill entry point, using its own `references/` directory).
That skill is NOT being restructured by this plan; its `references/` folder retains the
real files (not symlinks). These hits will **not** break and are out of scope.

---

## Bucket B — documentation references (no fix needed)

The following files mention the retiring filenames in the context of describing the
retirement itself. They do not route a reader to the retired file at runtime and will
not break.

| file | why it's safe |
|---|---|
| `docs/specs/2026-06-23-swiftui-reference-discoverability-design.md` (lines 47–53) | Spec describing the migration map — the filenames appear as migration targets, not as live skill routing |
| `docs/plans/2026-06-23-swiftui-reference-discoverability-plan.md` (lines 38, 59, 61, 71, 75–78, 85–87, 125) | Implementation plan — names files in task descriptions; updated as tasks complete, not a runtime consumer |
| `swiftui-specialist/evals/baseline.md` (lines 22, 36–44, 63–65, 79–82, 94, 96) | Baseline eval recording the pre-restructure dual-tree state; purely descriptive, not a routing instruction |
