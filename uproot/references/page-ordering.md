# Page Ordering Constructs

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

Access the current round number in templates: `{{ player.round }}`

Access data from previous rounds:
```python
player.within(round=1).get("choice")
player.within(app="my_app").along("round")  # iterate all rounds
```

## Repeat (conditional rounds)

Like Rounds but without a fixed count. `Repeat()` continues when the player has
`add_round = True` at the end of the block. Set it during one of the repeated
pages, usually in `before_once()` on the last page of the block:

```python
class Results(Page):
    @classmethod
    def before_once(page, player):
        player.add_round = player.round < C.NUM_ROUNDS

page_order = [
    Setup,
    Repeat(RoundInfo, Trade, Results),
]
```

See `call_auction` and `double_auction` for complete examples.

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

Check which branch was selected: `player.between_showed`

## Random (shuffle order)

Show all pages but in a randomized order:

```python
page_order = [
    Random(QuestionA, QuestionB, QuestionC),
    Results,
]
```

With brackets:
```python
page_order = [
    Random(
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
        Random(TaskA, TaskB, TaskC),
        ResultsPage,
        n=3,
    ),
    FinalResults,
]
```

## Randomizing entire apps

In `main.py`, use Bracket with splat to group an app's pages:

```python
from my_app1 import page_order as po1
from my_app2 import page_order as po2

page_order = [
    Random(Bracket(*po1), Bracket(*po2)),
    FinalPage,
]
```

See the `randomize_apps` directory for a complete example.
