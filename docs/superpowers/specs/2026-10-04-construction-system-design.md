# Real-time construction system

## Problem

Build mode (warehouse docks/racks/chargers/decor) and the Town planner (plots,
roads, houses, shops, offices) both apply a purchase instantly: pay credits,
the structure appears fully formed on the next frame. The player asked for
the game world to feel more "real" and to be built up "step by step" — the
first, highest-leverage piece of that is making construction take visible
time instead of being an instant credit-swap.

## Goals

- Every purchase in Build mode and the Town planner takes real (simulated)
  time to finish, with a visible rising structure instead of a pop-in.
- Builds are locked/non-functional until complete — this is a genuine
  gameplay tradeoff, not just presentation.
- Multiple builds can run in parallel, each on a short timer (well under one
  8-hour shift), so construction is a texture on top of the existing loop,
  not a new bottleneck to manage.
- Progress is driven by the existing simulated clock (`sim.sec`, scaled by
  `sim.speed`), so it respects pause/1×/3× and never depends on wall-clock
  time passing while the tab is closed.

## Non-goals (this spec)

- Ambience/presentation features (weather, sound, particles) — separate spec.
- Economic/logistics realism (fuel, traffic-driven ETAs, worker fatigue) —
  separate spec.
- A cancel-with-partial-refund button for in-progress warehouse builds.
- Making roads take construction time — they're infrastructure, not a
  "building," and gating pathfinding on a timer risks breaking routing for
  something orthogonal to this feature.

## Data model

A single persisted queue:

```
save.constr = [
  { id, site, kind:'wh'|'town', key, pos:[x,y,z], cost, startSec, dur, meta },
  ...
]
```

- `kind:'wh'` entries carry the same `key` shape `buildOptions()` already
  produces (`'bay'`, `'row'`, `'l3:2'`, `'fc:3'`, `'brk'`, `'decor:1'`).
- `kind:'town'` entries carry the tile key plus the deferred `{type, face,
  name, color}` that `applyTool()` currently writes to `T.b[t.key]`
  immediately — stashed in `meta` until completion.
- `startSec` is the absolute `sim.sec` at purchase time; `dur` is in
  sim-minutes. Progress = `clamp((sim.sec - startSec) / (dur*60), 0, 1)`.
  Because this is derived, not a running interval, it's safe across
  site-switches, saves/reloads, and tab closes.
- `dur = clamp(cost * 0.12, 5, 25)` sim-minutes — cheap items (25cr decor)
  finish in ~5 sim-minutes, expensive ones (160cr rack row) in ~20, all far
  shorter than one 480-minute shift.

## Warehouse Build mode

`buyBuild(key)` changes from applying the effect immediately to:

1. Validate cost/availability exactly as today.
2. Deduct credits, push a `save.constr` entry at the ghost's `pos`.
3. Leave that `buildOptions()` slot reserved (filtered out of the list, like
   today's already-owned items) so it can't be bought twice.
4. Spawn a scaffold object at `pos`: the existing ghost wireframe box, with
   the current translucent-fill-plus-"+"-sprite swapped for a radial
   progress-ring sprite (reusing the existing `ring` selection-ring visual
   language) that fills as `progress` advances.

No gameplay system currently reads "is this dock/rack usable" as a boolean
— they're usable because they exist in `L` (the computed layout). Since the
mutation to `L`'s source (`save.builds[code]`) doesn't happen until
completion, nothing has to change elsewhere to enforce "locked until done."

## Town planner

`applyTool()` changes for the building tools (not `buy`, not `bulldoze`, not
`road`):

1. Validate exactly as today (owns plot, plot is clear, has a road-adjacent
   face, affordable).
2. Deduct credits, set `t.type='constr'` (a new tile type: a small fenced
   lot + sign, rendered like other static tile props in `renderTile()`).
   This tile is not resellable, not counted as a road-connected income
   source, and not a valid delivery destination (`customers()` already
   filters by `BTYPES`/`type`, so a `constr` tile simply won't match).
3. Push a `save.constr` entry with `meta` holding the real `{type, face,
   name, color}` to apply on completion.

When the timer elapses, the tile flips from `'constr'` to the real type via
the same assignment `applyTool()` makes today (`t.type=meta.type`, etc.,
`T.b[t.key]=meta`), then `townChanged([key], roadCh)` runs as it does now to
refresh connectivity/income.

## Completion

A shared `tickConstr(dt)` (called from `tick()`, same cadence as other
per-frame simulation, not the throttled HUD batch) walks `save.constr`,
and for each entry whose progress reaches 1:

1. Applies the deferred mutation (the `buyBuild` logic for `kind:'wh'`, the
   tile-flip for `kind:'town'`).
2. Removes the scaffold/placeholder visual.
3. Refreshes that site's geometry — `loadSite()` if it's the currently
   detailed/run site, the existing lite-site rebuild path otherwise — and
   `townChanged()` for town tiles.
4. Toasts "Built: <label>" + confetti, matching today's instant-build
   feedback.
5. Removes the entry from `save.constr` and `writeSave()`.

## Undo

Ctrl+Z (`undoTown`) gains one new case: if the target tile is still
`'constr'`, undo cancels the matching `save.constr` entry, refunds the cost,
and reverts the tile — reusing the existing `townUndo` snapshot/restore
mechanism, just also clearing the queue entry. Once a build completes,
behavior is unchanged from today (bulldoze is the only way back).

## Testing

Non-trivial logic here is the progress/completion math and the
instant-vs-deferred mutation split. A `demo()`-style self-check (assert
progress clamps to [0,1], a `dur`-elapsed entry completes exactly once, an
undo on a `'constr'` tile fully refunds and reverts) covers the real risk —
no framework needed, matches the project's existing lack of a test
harness.
