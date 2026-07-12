# Page Types and Lifecycle Methods

## Page Types

### Page
Standard page that displays content and optionally collects form data.

```python
class MyPage(Page):
    fields = dict(
        name=StringField(label="Your name"),
    )
```

### NoshowPage
Hidden page that executes logic but is never displayed to the player. Use for
setup, scoring, randomization, and other behind-the-scenes computation.

```python
class ComputeScores(NoshowPage):
    @classmethod
    def after_always_once(page, player):
        player.score = compute(player)
```

### GroupCreatingWait
Waiting page that forms groups from arriving players. Players wait until enough
arrive to form a group of the specified size.

```python
class GroupPlease(GroupCreatingWait):
    group_size = 2

    @classmethod
    def after_grouping(page, group):
        for player, role in zip(group.players, [True, False]):
            player.is_first_mover = role
```

### SynchronizingWait
Waiting page where all group members must arrive before proceeding. Use for
calculating results after all players have made their decisions.

```python
class Sync(SynchronizingWait):
    @classmethod
    def all_here(page, group):
        total = sum(p.contribution for p in group.players)
        for p in group.players:
            p.payoff = cu(total * C.MPCR)
```

Session-level synchronization (waits for all players in the session, not just
the group):

```python
class WaitForEveryone(SynchronizingWait):
    synchronize = "session"

    @classmethod
    def all_here(page, session):
        create_groups(session, [list_of_player_groups])
```

Custom wait condition (override which players must arrive):

```python
class WaitForSubset(SynchronizingWait):
    @classmethod
    def wait_for(page, player):
        return [p.pid for p in player.group.players if p.is_active]
```

## Lifecycle Methods

Most page methods use `@classmethod` and receive `page` (the class) as their
first argument, followed by `player` (or `group`/`session` for wait pages).
Methods decorated with `@live` already become classmethods, so do not add an
extra `@classmethod` above them.

### Display control

```python
@classmethod
def show(page, player):
    """Return False to skip this page for this player."""
    return player.is_leader

@classmethod
def may_proceed(page, player):
    """Return False to prevent the player from advancing (e.g., in live pages)."""
    return player.trials_completed >= NUM_TRIALS
```

### Before-page hooks

```python
@classmethod
def before_once(page, player):
    """Runs only the first time the player visits this page."""
    player.trial_sequence = generate_trials()

@classmethod
def before_always_once(page, player):
    """Like before_once but also runs on NoshowPages."""
    pass
```

There is no `before_page` hook in uproot. Use `before_once` for visible pages
and `before_always_once` for logic that must also run on hidden/internal pages.

### Form fields

Fields can be a static dict or a dynamic method:

```python
# Static
fields = dict(
    amount=DecimalField(label="Amount", min=0, max=100),
)

# Dynamic (can be async)
@classmethod
async def fields(page, player):
    max_val = player.group.players.find_one(leader=True).offer * C.MULTIPLIER
    return dict(
        returned=DecimalField(label="How much?", min=0, max=max_val),
    )
```

### Validation

```python
@classmethod
def validate(page, player, data):
    """Return a string, list of strings, dict of field->error(s), or None."""
    if data.get("min_val") > data.get("max_val"):
        return "Minimum must be less than maximum."
    # Or field-specific errors:
    return {"min_val": "Too high", "max_val": ["Error 1", "Error 2"]}
```

### Stealth fields

Fields that are validated but NOT stored in the database. Use for comprehension
checks, passwords, or sensitive data you process but don't want persisted:

```python
stealth_fields = ["comprehension_check"]

@classmethod
def handle_stealth_fields(page, player, data):
    if data.get("comprehension_check") != "correct_answer":
        return "Please try again."
```

### Template data

```python
@classmethod
def jsvars(page, player):
    """Pass data to the template, accessible as uproot.vars in JS."""
    return {"trials": player.trial_data, "colors": COLORS}

# Template-only variables:
@classmethod
def templatevars(page, player):
    return {"options_a": [...], "options_b": [...]}
```

Do not define a `context` method; uproot rejects it because it was renamed to
`templatevars`.

### After-page hooks

```python
@classmethod
def after_once(page, player):
    """Runs once when the player successfully leaves this visible page."""
    player.total += player.contribution

@classmethod
def after_always_once(page, player):
    """Like after_once but also runs on NoshowPages and internal helper pages."""
    pass
```

There is no `before_next_page` hook in uproot. Put post-submission logic in
`after_once` or `after_always_once`. Wait pages should use `all_here`; uproot
rejects most `after_*` hooks on wait pages.

### Timeouts

```python
@classmethod
def timeout(page, player):
    """Return timeout in seconds. Page auto-submits when expired."""
    return 120  # 2 minutes

@classmethod
def timeout_reached(page, player):
    """Called when the timeout expires."""
    player.timed_out = True
```

For multi-page timeouts, store the deadline on the player and compute remaining
time dynamically:

```python
class Setup(NoshowPage):
    @classmethod
    def after_always_once(page, player):
        player.deadline = time() + 300  # 5 minutes total

class TaskPage(Page):
    @classmethod
    def timeout(page, player):
        return max(0, player.deadline - time())
```

### Live methods (WebSocket)

For real-time interaction without page reloads. Can be sync or async:

```python
@live
def increment(page, player):
    player.counter += 1
    return player.counter

@live
async def submit_answer(page, player, answer: str, reaction_time: float):
    correct = answer == player.expected
    return {"correct": correct, "done": player.trial >= NUM_TRIALS}
```

Call from JavaScript: `uproot.invoke("method_name", arg1, arg2)`

Combine with `may_proceed()` to control page advancement in live pages.

### Page properties

```python
allow_back = True  # Allow player to go back to this page
```
