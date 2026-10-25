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

Pass the player itself; identifier fields (`PlayerIdentifier`,
`SessionIdentifier`, `GroupIdentifier`, `ModelIdentifier`) are filled from the
entry's type annotations automatically:

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

For schemaless log lines, use `um.add_raw_entry(session.trials, {"event": "started"})`.

### Query entries

```python
for _, _, entry in um.filter_entries(session.trials, Trial, pid=player.pid):
    # entry.trial_number, entry.correct, etc.
    pass

# Count entries
count = sum(1 for _ in um.filter_entries(session.trials, Trial, pid=player.pid))
```

See the `custom_models` example.

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
directly: spawned tasks are supervised (exceptions are logged) and cancelled at
shutdown. Because they run outside page methods, wrap mutations in `with player:`
or `with session:`. Tasks do not survive restarts; re-create them in a
module-level `restart()`:

```python
async def restart() -> None:
    from uproot.storage import Admin

    with Admin() as admin:
        for session in admin.sessions:
            if session.get("counter") is not None:
                spawn(periodic_update(session))
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

Monitor for player disconnection. The handler must be a module-level `async`
function (it is stored by module and name); the default tolerance is 30 seconds:

```python
def new_player(player):
    watch_for_dropout(player, handle_dropout)

async def handle_dropout(player):
    player.dropout = True
    move_to_end(player)
```

`mark_dropout(player)` only adds the player to the manual dropout set for the
watcher; it does not call a handler or move the player by itself.

In multiplayer games, track dropouts at the **group** level: a `group.dropped`
flag set (once, guarded with `if not group.get("dropped")`) by the dropout
handler, by `timeout_reached` on decision pages, and by a `timeout` on the
`SynchronizingWait`. Then `move_to_page` the other members to a terminal
`Dropped` page, guard later pages with `before_once`, and skip dropped groups in
`all_here`, `digest`, and `pipeline`. Register watchers in `after_grouping`
(inside `with player:`), not in `new_player`, when the handler needs the group.
Share the repeated timeout/guard methods via a mixin class
(`class Dilemma(DroppableMixin, Page)`) instead of copying them onto every page.
The uproot-docs page `docs/advanced/dropouts.md` has the complete pattern; see
also the `dropouts` example.

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

Both accept a session object or a `SessionIdentifier`.

## Randomization Utilities

Use `rng()` for experiment randomization. It supplies a separately seeded
`random.Random` instance. Do not use Python's process-global random generator.

```python
items = list(item_list)
rng().shuffle(items)

# A stream that continues across pages: store the generator itself
player.rng = rng()
player.bomb = player.rng.randint(1, 25)

# Random page ordering
page_order = [RandomOrder(PageA, PageB, PageC)]
```

`uuid()` returns a fresh UUID (uuid7 where available). See the `rerandomization`
example for on-the-fly stratified assignment.

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

Declare defaults in `main.py` with `load_config(..., settings={...})` and read
them with the built-in helper. Treat settings as read-only, and always pass a
default, since the automatic single-app configs (`~my_app`) have no settings:

```python
class Setup(NoshowPage):
    @classmethod
    def after_always_once(page, player):
        player.duration = get_setting(player.session, "duration", 120)
```

`get_setting` is also available in templates.

### Custom settings forms (`AdminSettings.html`)

An app may replace the admin's raw JSON settings editor with its own form by
adding an `AdminSettings.html` fragment (no `extends`). It receives `config`,
`apps`, `appname`, `settings` (the config defaults), `editor_id`, `C`, and
`appstatic`, and must register a reader that validates the inputs and returns
the settings to merge:

```html
{% set n_rounds = settings.get("n_rounds", C.DEFAULT_ROUNDS) %}

<div id="{{ editor_id }}">
    <label class="form-label" for="{{ editor_id }}-rounds">Number of rounds</label>
    <input class="form-control" id="{{ editor_id }}-rounds" type="number"
        min="1" step="1" value="{{ n_rounds }}">
</div>

<script>
registerSessionSettingsEditor("{{ editor_id }}", () => {
    const rounds = Number(document.getElementById("{{ editor_id }}-rounds").value);

    if (!Number.isInteger(rounds) || rounds < 1) {
        throw new TypeError("Number of rounds must be a positive integer.");
    }

    return { n_rounds: rounds };
});
</script>
```

Derive every element ID from `editor_id`: the fragment may render several times
on one page. Throwing blocks session creation and shows the message. A
project-root `AdminSettings.html` replaces all app fragments. Only build this when
the user wants it; the JSON editor is fine for most apps. See `call_auction`,
`double_auction`, `flow_divider`, and `message_board`.

## Digest and Pipeline (Admin Reporting)

`digest(session)` returns a dict whose keys become variables in the app's
`AdminDigest.html`, a live summary for the experimenter:

```python
def digest(session: SessionType) -> dict[str, Any]:
    choices = [
        p.within(app=__name__).get("choice") for p in session.players
    ]
    answered = [c for c in choices if c is not None]

    return {"n_total": len(answered), "n_yes": sum(answered)}
```

```html
{# AdminDigest.html #}
{% block main %}
<p>Responses so far: <b>{{ n_total }}</b> ({{ n_yes }} yes)</p>
{% endblock main %}
```

`pipeline(session, data=None)` returns rows that the admin can run and download
as a table. `data` is posted JSON or `None`; an optional `AdminPipeline.html` can
provide the form that posts it:

```python
def pipeline(session: SessionType, data: Any = None) -> list[dict[str, Any]]:
    rows = []

    for player in session.players:
        for round_number, r in player.within(app=__name__).along("round"):
            rows.append({
                "player": player.name,
                "round": round_number,
                "choice": r.get("choice"),
                "payoff": r.get("payoff"),
            })

    return rows
```

If the app has several blocks, narrow with `within(app=__name__, block=...)`. Use
`Context(player)`, not `player.context`, inside both functions. See `mind_game`
and `call_auction` for both callbacks.

## App HTTP APIs

`api(request, session)` serves `/api/{app}/{session}/` and requires an
`Authorization: Bearer` token listed in `upd.API_KEYS`; use it for trusted
scripts. `api2(request, session, player=None)` serves `/api2/{app}/{session}/`
for participant browsers. Callbacks receive keyword arguments, and `player` is
only passed when the request came from `uproot.api2()` on the participant's page:

```python
from fastapi import HTTPException

async def api2(request, session, player=None):
    if player is None:
        raise HTTPException(status_code=403, detail="Participant required")

    data = await request.json()
    player.last_action = data["action"]

    return {"saved": True}
```

Never identify participants from query strings or bodies; use only the
authenticated `player`. Return a dict (sent as JSON) or a Starlette `Response`
(see the `captcha` example).

## Internationalization

Built-in strings ship in `en`, `de`, `es`, `fr`, and `ja`; set the project
default with `upd.LANGUAGE = "de"` in `main.py`. Load your own YAML translations
(one file per language, e.g. `locales/fr.yml`, keyed by the exact template
phrase) and choose a language per participant:

```python
import uproot.i18n as i18n

i18n.load("my_app/locales/")

def language(player):
    return player.get("language") or "en"
```

In templates use `_("text")` or `{% translate %}...{% endtranslate %}`; in
JavaScript `_("text")`. Keys must match exactly, including straight vs.
typographical quotes. Run `uv run uproot check-translations` to list missing
phrases (it also checks field labels, descriptions, and choices). See the
`multilanguage` example.

## Utility Functions

These are all exported from `uproot.smithereens`:

- `get_setting(session, key, default=None)` - read admin-configured session settings
- `rng()` - independent `random.Random` seeded from the OS
- `uuid()` - fresh UUID
- `safe(html_string)` - mark a string as HTML-safe (equivalent to `Markup`)
- `data_uri(data: bytes)` - encode binary data as a `data:` URI
- `read_csv(path)` - read a CSV file into a list of dicts
- `append_to_csv(path, data)` - append a dict as a row to a CSV file
- `reload(player)` - force-reload a player's browser page
- `spawn(coro)` - schedule an async coroutine as a background task
- `move_to_page(player, page_class)` / `transition_to_page(...)` - move a player
  forward to a page (aliases; see `references/page-ordering.md`)
- `move_to_end(player)` / `transition_to_end(player)` - move a player to the end
- `add_to_group(group, player)` - add a player to an existing group
- `identify(obj)` - get the identifier for a player/group/session/model
- `fmtnum` - number formatter for Python and templates (`| fmtnum(...)`)
