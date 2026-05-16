# Game Theory Experiment Patterns

## Standard 2-Player Game Structure

Most 2-player games follow this flow:

```
GroupCreatingWait → Decision(s) → SynchronizingWait → Results
```

### Simultaneous decisions (e.g., Prisoner's Dilemma, Cournot)

Both players decide at the same time, then results are revealed:

```python
DESCRIPTION = "My game"
SUGGESTED_MULTIPLE = 2

class C:
    ENDOWMENT = cu("10")

class GroupPlease(GroupCreatingWait):
    group_size = 2

class Decision(Page):
    fields = dict(
        choice=RadioField(label="Your choice", choices=[(True, "A"), (False, "B")]),
    )

class Sync(SynchronizingWait):
    @classmethod
    def all_here(page, group):
        p1, p2 = group.players
        # Calculate payoffs based on both choices
        p1.payoff = ...
        p2.payoff = ...

class Results(Page):
    pass

page_order = [GroupPlease, Decision, Sync, Results]
```

### Sequential decisions (e.g., Trust Game, Stackelberg, Ultimatum)

One player acts first, the other responds:

```python
class GroupPlease(GroupCreatingWait):
    group_size = 2

    @classmethod
    def after_grouping(page, group):
        for player, is_first in zip(group.players, [True, False]):
            player.is_first_mover = is_first

class FirstMoverDecision(Page):
    fields = dict(offer=DecimalField(label="Your offer", min=0, max=C.ENDOWMENT))

    @classmethod
    def show(page, player):
        return player.is_first_mover

class WaitForFirst(SynchronizingWait):
    pass

class SecondMoverDecision(Page):
    @classmethod
    def show(page, player):
        return not player.is_first_mover

    @classmethod
    def fields(page, player):
        first = player.group.players.find_one(is_first_mover=True)
        return dict(
            response=DecimalField(label="Your response", min=0, max=first.offer * C.MULTIPLIER),
        )

class Sync(SynchronizingWait):
    @classmethod
    def all_here(page, group):
        first = group.players.find_one(is_first_mover=True)
        second = group.players.find_one(is_first_mover=False)
        first.payoff = C.ENDOWMENT - first.offer + second.response
        second.payoff = first.offer * C.MULTIPLIER - second.response

class Results(Page):
    pass

page_order = [GroupPlease, FirstMoverDecision, WaitForFirst, SecondMoverDecision, Sync, Results]
```

### Payoff matrices (e.g., Prisoner's Dilemma, 2x2 games)

```python
class C:
    PAYOFF_MATRIX = {
        (True, True): (3, 3),     # Both cooperate
        (True, False): (0, 5),    # I cooperate, they defect
        (False, True): (5, 0),    # I defect, they cooperate
        (False, False): (1, 1),   # Both defect
    }

class Sync(SynchronizingWait):
    @classmethod
    def all_here(page, group):
        p1, p2 = group.players
        payoffs = C.PAYOFF_MATRIX[(p1.choice, p2.choice)]
        p1.payoff = cu(payoffs[0])
        p2.payoff = cu(payoffs[1])
```

## N-Player Games (e.g., Public Goods, Beauty Contest)

```python
class C:
    GROUP_SIZE = 4
    ENDOWMENT = cu("20")
    MPCR = cu("0.4")

class GroupPlease(GroupCreatingWait):
    group_size = C.GROUP_SIZE

class Context(PlayerContext):
    @property
    def total(self):
        return sum(p.contribution for p in self.player.group.players)

class Sync(SynchronizingWait):
    @classmethod
    def all_here(page, group):
        total = sum(p.contribution for p in group.players)
        for p in group.players:
            p.payoff = C.ENDOWMENT - p.contribution + total * C.MPCR
```

## Repeated Games

Wrap the decision-results cycle in `Rounds()`:

```python
page_order = [
    GroupPlease,
    Rounds(Decision, Sync, Results, n=C.ROUNDS),
    FinalResults,
]
```

Access historical data in templates:
```html
{% for round in range(1, player.round) %}
    <tr>
        <td>{{ round }}</td>
        <td>{{ player.within(round=round).get("choice") }}</td>
    </tr>
{% endfor %}
```

### Digest function

For admin reporting across rounds:

```python
def digest(session):
    rows = []
    for player in session.players:
        for round in range(1, C.ROUNDS + 1):
            p = player.within(round=round)
            rows.append({
                "player": player.name,
                "round": round,
                "choice": p.get("choice"),
                "payoff": p.get("payoff"),
            })
    return rows
```

## Treatment Assignment

```python
TREATMENTS = [1, 2, 3]

def new_player(player):
    player.treatment = rng().choice(TREATMENTS)
```

For balanced assignment, follow the `treatments_balanced` example and assign the
least-used treatment among the players you want to balance over:
```python
from collections import Counter

class AssignTreatment(NoshowPage):
    @classmethod
    def after_always_once(page, player):
        counts = Counter(player.session.players.apply(lambda p: p.get("treatment")))
        player.treatment = min(TREATMENTS, key=lambda x: counts.get(x, 0))
```

## Currency

Use `cu()` for monetary values:
```python
class C:
    ENDOWMENT = cu("10")
    EXCHANGE_RATE = cu("0.5")

# In calculations:
player.payoff = cu(10) - player.offer
player.payoff = C.ENDOWMENT * C.MULTIPLIER
```

## Context Properties

Use `PlayerContext` for computed values that depend on group state:

```python
class Context(PlayerContext):
    @property
    def partner_choice(self):
        return self.player.other_in_group.choice

    @property
    def total_contributions(self):
        return sum(p.contribution for p in self.player.group.players)

    @property
    def payoff_info(self):
        matrix = C.PAYOFF_MATRIX
        me = self.player.choice
        other = self.player.other_in_group.choice
        return matrix[(me, other)]
```

Access in templates: `{{ player.context.partner_choice }}`

## Existing Game Examples

Before building a new game, check these examples in the repository:
- `dictator_game` - Dictator game (simplest 2-player)
- `ultimatum_game` - Ultimatum game (sequential, accept/reject)
- `trust_game` - Trust game (sequential, send/return)
- `prisoners_dilemma` - Prisoner's dilemma (simultaneous, payoff matrix)
- `prisoners_dilemma_repeated` - Repeated PD with history
- `prisoners_dilemma_chat` - PD with chat
- `public_goods_game` - Public goods (N-player)
- `beauty_contest` - Guessing game (N-player)
- `cournot` - Cournot competition (simultaneous)
- `bertrand` - Bertrand competition (simultaneous)
- `stackelberg` - Stackelberg competition (sequential)
- `gift_exchange_game` - Gift exchange (sequential, roles)
- `focal_point` - Coordination game
- `travellers_dilemma` - Traveller's dilemma
- `minimum_effort_game` - Weakest link (N-player)
- `twobytwo` - Generic 2x2 game with configurable matrix
- `call_auction` - Call auction (advanced, with models)
- `double_auction` - Double auction (advanced, real-time)
