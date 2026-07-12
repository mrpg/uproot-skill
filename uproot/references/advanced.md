# Advanced Patterns

## Persistent Data Models (uproot.models)

For data that doesn't fit as simple player attributes (e.g., per-trial
measurements, order books, transaction logs, variable-length records):

### Define an Entry class

```python
import uproot.models as um

class Trial(metaclass=um.Entry):
    pid: PlayerIdentifier
    trial_number: int
    word: str
    response: str
    correct: bool
    reaction_time_ms: float
```

### Initialize the model in new_session

```python
def new_session(session):
    session.trials = um.create_model(session, tag="trials")
```

### Store entries

```python
um.add_entry(
    session.trials,
    player,
    Trial,
    trial_number=1,
    word="RED",
    response="blue",
    correct=False,
    reaction_time_ms=423.7,
)
```

### Query entries

```python
for _, _, entry in um.filter_entries(session.trials, Trial, pid=player.pid):
    # entry.trial_number, entry.correct, etc.
    pass

# Count entries
count = sum(1 for _ in um.filter_entries(session.trials, Trial, pid=player.pid))
```

### Use cases

- Stroop task: store each trial's response and reaction time
- Auctions: store bids, asks, and transactions
- Conjoint: store profile pairs and preferences
- Any task with variable-length output per player

## Live Methods (@live)

Real-time server communication without page reloads:

### Basic pattern

Live methods can be sync or async:

```python
class TaskPage(Page):
    @classmethod
    def may_proceed(page, player):
        return player.done

    @live
    def get_state(page, player):
        return {"current_trial": player.current, "total": NUM_TRIALS}

    @live
    def submit_answer(page, player, answer: str, rt: float):
        correct = answer == player.expected
        player.current += 1
        player.done = player.current >= NUM_TRIALS
        return {"correct": correct, "done": player.done}
```

### Client-side

```javascript
// Call a live method
uproot.invoke("get_state").then(state => {
    console.log(state.current_trial);
});

// With arguments
uproot.invoke("submit_answer", userAnswer, reactionTime).then(result => {
    if (result.done) {
        uproot.submit();  // Advance to next page
    }
});
```

### With Alpine.js

```html
<div x-data="taskApp()" x-init="init()">
    <div x-show="!done">
        <p x-text="prompt"></p>
        <button @click="respond('A')">A</button>
        <button @click="respond('B')">B</button>
    </div>
    <div x-show="done">
        <p>Complete! Click Next to continue.</p>
    </div>
</div>

<script>
function taskApp() {
    return {
        prompt: '', done: false,
        async init() {
            const state = await uproot.invoke("get_state");
            this.prompt = state.prompt;
        },
        async respond(choice) {
            const result = await uproot.invoke("submit_answer", choice);
            if (result.done) {
                this.done = true;
                uproot.submit();
            } else {
                const state = await uproot.invoke("get_state");
                this.prompt = state.prompt;
            }
        }
    };
}
</script>
```

## Server-to-Client Notifications

Push data from server to specific players or groups:

```python
# Notify the other player in a 2-player group
notify(player, player.other_in_group, {"type": "update", "value": 42})

# Notify a single player by reference
send_to_one(target_player, data={"type": "update"})

# Broadcast to all session players
send_to(session.players, data={"type": "refresh"})

# Force a player to reload their page
reload(player)
```

Client-side receiving:
```javascript
uproot.receive = (data) => {
    if (data.type === "update") {
        document.getElementById("value").textContent = data.value;
    }
};
```

Use `event` when you want a custom browser event instead of `uproot.receive`:

```python
notify(player, player.session.players, data, event="Notified", where=...)
```

Listen for custom events:
```javascript
uproot.onCustomEvent("Notified", (event) => {
    // event.detail.data contains the payload
});
```

## Background Tasks

Run async tasks that continue independently of page loads using `spawn`:

```python
import asyncio

async def periodic_update(session):
    while True:
        with session:
            session.counter += 1
            send_to(session.players, data={"counter": session.counter})

        await asyncio.sleep(5)

class Setup(NoshowPage):
    @classmethod
    def after_always_once(page, player):
        if player.session.get("counter") is None:
            player.session.counter = 0
            spawn(periodic_update(player.session))
```

Use `spawn()` (from `uproot.smithereens`) instead of `asyncio.create_task()`
directly. For tasks that must survive server restarts, define an `async def
restart()` callback at module level to re-create them.

## Chat Integration

### Setup

```python
def new_session(session):
    session.chat = chat.create(session)

class ChatPage(Page):
    @classmethod
    def before_once(page, player):
        chat.add_player(player.session.chat, player, pseudonym=f"Player {player.id}")
```

### Optional message callback

```python
def on_chat_message(session_chat, player, message):
    # Process message, e.g., log it
    pass

def new_session(session):
    session.chat = chat.create(session)
    chat.on_message(session.chat, on_chat_message)
```

### Template

```html
{{ chat(session.chat) }}
```

## Dropout Handling

Monitor for player disconnection:

```python
def new_player(player):
    watch_for_dropout(player, handle_dropout)

async def handle_dropout(player):
    player.dropout = True
    move_to_end(player)
```

Mark a player as dropped without a callback: `mark_dropout(player.pid)`

## Custom Group Creation

For more control than `GroupCreatingWait`:

```python
class WaitForEveryone(SynchronizingWait):
    synchronize = "session"

    @classmethod
    def all_here(page, session):
        players = sorted(session.players, key=lambda p: p.name)
        mid = len(players) // 2
        group1 = players[:mid]
        group2 = players[mid:]
        create_groups(session, [group1, group2])
```

Or with `create_group()` for a single group:
```python
create_group(session, players_list)
```

## Randomization Utilities

Use `rng()` for experiment randomization. It supplies a separately seeded
`random.Random` instance. Do not use Python's process-global random generator.

```python
items = list(item_list)
rng().shuffle(items)

# Random page ordering (Random is exported from uproot.smithereens)
page_order = [Random(PageA, PageB, PageC)]
```

Store any assigned treatment, order, or draw on the player before it affects the
participant flow, so it remains stable on refresh and can be exported.

## Stealth Fields (Sensitive Data)

Fields processed but not stored in the database:

```python
class PaymentPage(Page):
    stealth_fields = ["iban"]

    fields = dict(
        iban=IBANField(label="Your IBAN"),
    )

    @classmethod
    async def handle_stealth_fields(page, player, data):
        iban = data.get("iban")
        # Process IBAN (e.g., send to payment provider)
        # It will NOT be stored in the uproot database
```

## File Uploads

```python
class UploadPage(Page):
    fields = dict(
        document=FileField(label="Upload your file"),  # FileField is always handled as stealth
    )

    @classmethod
    async def handle_stealth_fields(page, player, data):
        uploaded = data.get("document")  # UploadFile object
        content = await uploaded.read()
        # Process file content
```

## Hashed Answers (Anti-Cheating)

For quizzes where answers should not be visible in HTML source:

```python
from uproot.types import sha256

QUIZ = [
    ("What is 2+2?", ["4", "3", "5"]),
]

class QuizPage(Page):
    stealth_fields = [f"q{i}" for i, _ in enumerate(QUIZ)]

    @classmethod
    def before_once(page, player):
        player.quiz_choices = [
            rng().sample(answers, k=len(answers))
            for _, answers in QUIZ
        ]

    @classmethod
    def fields(page, player):
        result = {}
        for i, (question, answers) in enumerate(QUIZ):
            choices = [(sha256(a), a) for a in player.quiz_choices[i]]
            result[f"q{i}"] = RadioField(label=question, choices=choices)
        return result

    @classmethod
    def handle_stealth_fields(page, player, data):
        for i, (question, answers) in enumerate(QUIZ):
            correct_hash = sha256(answers[0])  # First answer is correct
            if data.get(f"q{i}") != correct_hash:
                return f"Wrong answer for: {question}"
```

## Session Settings

Read configurable parameters set in the admin interface with the built-in
helper. Treat settings as read-only:

```python
class Setup(NoshowPage):
    @classmethod
    def after_always_once(page, player):
        player.duration = get_setting(player.session, "duration", 120)
```

## Digest Functions (Admin Reporting)

Aggregate data for the admin's data export:

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

## Internationalization

```python
from uproot.i18n import load as load_all

load_all("my_app/")

def language(player):
    return player.language or "en"

def new_player(player):
    player.language = "en"
```

See the `multilanguage` example for the full pattern with language switching.

## Utility Functions

These are all exported from `uproot.smithereens`:

- `get_setting(session, key, default=None)` - read admin-configured session settings
- `safe(html_string)` - mark a string as HTML-safe (equivalent to `Markup`)
- `data_uri(data: bytes)` - encode binary data as a `data:` URI
- `read_csv(path)` - read a CSV file into a list of dicts
- `append_to_csv(path, data)` - append a dict as a row to a CSV file
- `reload(player)` - force-reload a player's browser page
- `spawn(coro)` - schedule an async coroutine as a background task
- `transition_to_page(player, page_class)` - move player to a specific page
- `transition_to_end(player)` - move player to the end of the experiment
- `move_to_page(player, page_class)` - move player to a page (legacy alias)
- `move_to_end(player)` - move player to the end (legacy alias)
- `add_to_group(group, player)` - add a player to an existing group
- `identify(obj)` - get the identifier for a player/group/session/model
- `fmtnum` - number formatter for Python and templates (`| fmtnum(...)`)
