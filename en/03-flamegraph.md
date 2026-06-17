# Reading a flamegraph

A flamegraph is a picture of **where your program spent its time**. It takes a
little practice to read, so this chapter starts from zero and goes slowly.

## What the picture actually is

While measuring, prperf (rperf) takes an **enormous number of snapshots** of
your program and records "what was running at that moment" (the call stack).
Stacking thousands of those photos and tallying them gives the flamegraph.

- **One box = one method** (function).
- **The width of a box = the share of time spent in that method** (and whatever
  it called) — i.e. how many photos caught it running. **Wider = heavier.**
- **The vertical stacking = call depth.** The bottom is the entry point; the
  higher you go, the closer to the **leaf** (what was actually executing).

A picture beats words. Here is an example (bottom = entry, top = leaf):

```
          ┌──────────┐
          │ String#* │                      ← leaf: actually on the CPU (wider = heavier)
      ┌───┴──────────┴───┐
      │  JSON.generate   │                  ← its caller
   ┌──┴──────────────────┴────────────┐
   │          Integer#times           │     ← a loop: wide, but it just calls its children
┌──┴──────────────────────────────────┴──┐
│                 <main>                 │  ← the whole program; bottom = root, always ~100% wide
└────────────────────────────────────────┘
```

What this tells you:

- The bottom `<main>` is 100% wide. That's the **whole program**, so it is
  expected — don't be alarmed by it.
- The real cost is in the **wide boxes near the top** (here `String#*`). That is
  where the CPU was actually pinned.

## The two rules that matter most

1. **Only look at width.** Width = cost. Read each box as "if I could make this
   zero, that's how much I'd save."
2. **Ignore height.** A tall tower (deep nesting) is cheap if it's thin. A short
   box is heavy if it's wide. **Hunt for wide plateaus, not tall towers.**

## Common misreadings

- **Left-to-right is NOT time order.** It is not "this ran, then that"; boxes are
  just sorted (e.g. by name). Don't read it as a timeline.
- **Color means nothing in the single view.** It only separates boxes; "red" is
  not "hot." (Color *does* mean something in **diff mode** — see below.)
- **A 100%-wide box at the bottom is normal.** It just represents everything.

## A reading recipe

1. Find the **wide boxes near the top** — that's where your time goes.
2. **Click to zoom** into that box: its subtree fills the width, so the
   breakdown is easier to see.
3. If a method name interests you, **search** for it to highlight it across the
   whole graph — even if it's scattered, the combined width reveals the cost.
4. Prefer a table? Use the **Top tab**: methods listed by self / cumulative
   weight. Work down from the top.

## In prperf you mostly use "diff mode"

Opening the diff link from the Check Run puts the viewer in **diff mode** — this
is the heart of regression hunting.

- **Now color has meaning.** Compared to base, **methods whose share increased
  are red, decreased are blue**, and near-unchanged are neutral.
- So in diff mode you look for the **reddest** box, not the widest. That is
  "what got heavier in this PR."
- How to read it:
  - **Wide and red** = something already significant that got worse. Top
    priority.
  - **Narrow but bright red** = newly appeared, or grew sharply in share. A likely
    new culprit.
  - Remember: width = head's share, color = the change from base.

## Handy viewer controls

- **Click** — zoom into a box (its subtree fills the width).
- **Search box** — filter/highlight by name; type `JSON` to grasp all related
  spots at once.
- **Top / Tags tabs** — for table lovers; sorted by self / cumulative weight.
- **Shift+click** — pin a method; a sparkline shows its share across snapshots
  (time travel).
- **j / k** — move to newer / older snapshots.

## Note: this is a picture of *time*

prperf's flamegraph is a picture of **time (CPU weight)**. Allocation and GC
counts live in the **summary numbers** (the Check Run table). For an "alloc went
up" regression, use the flamegraph to find the method whose **time grew (turned
red)**, then check whether that path is creating extra objects.

## Summary (all you really need)

- Width = cost; ignore height.
- Wide near the top = where your time goes.
- Left-to-right is not order; single-view color is meaningless.
- In prperf's diff, **find the red box** — that's what got worse this time.
