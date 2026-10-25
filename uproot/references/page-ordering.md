# Page Ordering Constructs

The operators below ("SmoothOperators") are exported from `uproot.smithereens`.
They accept **only pages and other SmoothOperators**. A list such as
`RandomOrder([A, B], [C, D])` raises a `TypeError`; use `Bracket(A, B)` instead.
A plain `page_order` list may still contain nested lists; they are flattened.

Most operator constraint violations (lists inside operators, non-integer `n`,
missing blocks) are reported when `uproot run` starts. A `page_order` *function*
is only checked when a participant joins, so test such apps with a participant.

## Basic sequence

```python
page_order = [Page1, Page2, Page3]
```

## Rounds

Repeat a block of pages n times:

```python
page_order = [
    Instructions,
    Rounds(Decision, Sync, Results, n=5),
    FinalResults,
]
```

`n` must be an `int` of 0 or more: write `n=len(C.ITEMS) // 2`, never `/ 2`.
`Rounds(..., n=0)` never runs.

Access the current round number (1-indexed) in templates: `{{ player.round }}`

Access data from previous rounds:
```python
player.within(round=1).get("choice")
player.within(app=__name__).along("round")  # iterate (round, data) pairs
player.within(round=3, strict=True).get("choice")  # only values written in round 3
```

Without `strict=True`, a view may carry forward a value stored before that round.

When the participant leaves the app, uproot removes `player.round`,
`player.block`, and `player.round_nested`. Reading `player.round` in a later app
without its own loop raises `AttributeError`; use `player.within(app=...,
round=...)` to look back.

## Repeat (conditional rounds)

Like Rounds but without a fixed count. `Repeat()` continues when the player has
`add_round = True` at the end of the block. Set it during one of the repeated
pages, usually in `before_once()` on the last page of the block:

```python
class Results(Page):
    @classmethod
    def before_once(page, player: PlayerType) -> None:
        player.add_round = player.round < C.NUM_ROUNDS

page_order = [
    Setup,
    Repeat(RoundInfo, Trade, Results),
]
```

See `call_auction` and `double_auction` for complete examples.

## Blocks (several loops in one app)

If participants can go through **more than one** `Rounds` or `Repeat` in the same
app, every outermost loop in that app needs a `block=` name, or `uproot run`
refuses to start:

```python
page_order = [
    Instructions,
    Rounds(Decision, Feedback, n=2, block="practice"),
    Rounds(Decision, Feedback, n=10, block="main"),
    FinalResults,
]
```

- Each block counts `player.round` from 1; `player.block` holds the block name
  (`None` for an unnamed loop). Block names must be valid Python identifiers.
- Loops with the **same** block name share one counter and continue where they
  left off (`Rounds(A, n=2, block="red")`, ..., `Repeat(A, block="red")` gives red
  rounds 1, 2, then 3, 4, ...). Counters restart in every app.
- Loops inside `Between` count once (each participant sees only one alternative),
  and `n=0` loops are ignored. `RandomOrder(Rounds(...), Rounds(...))` runs both
  loops, so both need blocks.
- A loop nested inside another belongs to the outer block and cannot name its
  own. `player.round` keeps counting through all levels; `player.round_nested`
  lists the position per level (e.g., `[2, 1]`).
- Narrow history queries by block, or rounds of different blocks get mixed:
  `player.within(app=__name__, block="main").along("round")`.

See the `blocks` example.

## Between (random branch)

Randomly select ONE page (or bracketed group) to show:

```python
page_order = [
    Between(TreatmentA, TreatmentB, TreatmentC),
    Results,
]
```

Use `Bracket()` to group multiple pages as a single branch:

```python
page_order = [
    Between(
        PageA,                         # Branch 1: single page
        Bracket(PageB, PageC),         # Branch 2: two pages shown together
        Bracket(PageD, PageE, PageF),  # Branch 3: three pages shown together
    ),
    Results,
]
```

`player.between_showed` is a list with one entry per `Between` the participant
went through. Each entry lists the pages of the selected branch as
`"app/PageName"`:

```python
treated = f"{__name__}/TreatmentTask" in player.between_showed[-1]
```

A nested inner `Between` adds its own entry (the outer entry then omits its
pages), and a branch without pages gets `[]`. For treatment assignment that you
analyze later, prefer storing an explicit `player.treatment` (see
`references/advanced.md`) over parsing `between_showed`.

## RandomOrder (shuffle order)

Show all pages but in a randomized order. (`Random` was renamed to
`RandomOrder`; `Random` no longer exists.)

```python
page_order = [
    RandomOrder(QuestionA, QuestionB, QuestionC),
    Results,
]
```

With brackets:
```python
page_order = [
    RandomOrder(
        Bracket(Intro1, Task1),
        Bracket(Intro2, Task2),
        Bracket(Intro3, Task3),
    ),
    Results,
]
```

See `randomize_pages` and `randomize_pages_allow_back` examples.

## Nesting

Constructs can be nested:

```python
page_order = [
    Rounds(
        RandomOrder(TaskA, TaskB, TaskC),
        ResultsPage,
        n=3,
    ),
    FinalResults,
]
```

## Randomizing entire apps

Use Bracket with splat to group an app's pages:

```python
from . import app1, app2

page_order = [
    RandomOrder(Bracket(*app1.page_order), Bracket(*app2.page_order)),
    FinalPage,
]
```

See the `randomize_apps` directory for a complete example.

## Moving players programmatically

`move_to_page(player, Target)` (alias of `transition_to_page`) moves a player
**forward** to the next occurrence of `Target`. Operators on the way are set up
as if the player had walked there (the target's round starts, `RandomOrder`
shuffles, `Between` selects). It raises `ValueError`, leaving the player
unchanged, if the target is not ahead or a `Between` on the way did not select
it. `move_to_end(player)` (alias of `transition_to_end`) skips to the end and
runs the end page's hook. Both reload the player's browser by default.
