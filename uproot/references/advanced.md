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

```python
class TaskPage(Page):
    @classmethod
    def may_proceed(page, player):
        return player.done

    @live
    async def get_state(page, player):
        return {"current_trial": player.current, "total": NUM_TRIALS}

    @live
    async def submit_answer(page, player, answer: str, rt: float):
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
from uproot.smithereens import notify

# Notify the other player in a 2-player group
notify(player, player.other_in_group, {"type": "update", "value": 42})

# In the client
uproot.receive = (data) => {
    if (data.type === "update") {
        document.getElementById("value").textContent = data.value;
    }
};
```

For session-wide broadcast:
```python
from uproot.smithereens import send_to

def broadcast(session, data):
    send_to(session.players, data=data)
```

Use `event` when you want a custom browser event instead of `uproot.receive`:

```python
notify(player, player.session.players, data, event="Notified", where=...)
```

## Background Tasks

Run async tasks that continue independently of page loads:

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
            asyncio.create_task(periodic_update(player.session))
```

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
from uproot.smithereens import watch_for_dropout, move_to_end

def new_player(player):
    watch_for_dropout(player, handle_dropout)

async def handle_dropout(player):
    player.dropout = True
    move_to_end(player)
```

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

```python
from random import Random

from uproot.smithereens import Random as RandomPages


def shuffled(iterable, *, seed=None):
    result = list(iterable)
    Random(seed).shuffle(result)
    return result

# Deterministic shuffle (seeded by player)
items = shuffled(item_list, seed=hash(player.name + "task"))

# Random page ordering
page_order = [RandomPages(PageA, PageB, PageC)]
```

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
    def fields(page, player):
        result = {}
        for i, (question, answers) in enumerate(QUIZ):
            choices = [(sha256(a), a) for a in shuffled(answers, seed=hash(player.name + f"q{i}"))]
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

Read configurable parameters set in the admin interface:

```python
def get_setting(session, key, default):
    if session.settings and key in session.settings:
        return session.settings[key]
    return default

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

def new_player(player):
    player.language = "en"
```

See the `multilanguage` example for the full pattern with language switching.
