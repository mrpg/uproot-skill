---
name: uproot
description: >
  Build browser-based behavioral experiments with the uproot framework. Use this
  skill whenever the user asks to create, modify, or debug an uproot app/experiment,
  mentions "uproot" in the context of experiments, wants to build a game theory
  experiment (dictator game, public goods, trust game, ultimatum, prisoner's dilemma,
  auction, etc.), needs a survey/questionnaire, real effort task, or any interactive
  multiplayer study. Also activate when the user references uproot page types
  (Page, NoshowPage, GroupCreatingWait, SynchronizingWait), uproot fields
  (DecimalField, RadioField, LikertField, etc.), uproot imports (uproot.smithereens,
  uproot.fields, uproot.models), or asks about rounds, groups, live/WebSocket
  features, or simulate.js in an experiment context.
---

# Building Uproot Experiments

uproot is a Python framework for browser-based behavioral experiments. Apps are
self-contained directories with Python logic, Jinja2 HTML templates, and optional
JavaScript. This skill guides you through building them correctly.

## Converting from Other Platforms

When asked to convert an experiment from **Qualtrics**, **oTree**, or other well-known
platforms, work directly from the provided source code — no need to ask for additional
documentation or platform-specific guidance.

You must NEVER attempt to implement a "compatibility layer", or shims, or similar, when
converting experiments. You MUST ALWAYS reimplement from scratch using best practices!

For **z-Tree** treatment files (`*.ztt`), these are binary files that must be unpacked
first. Use the Python tool at https://github.com/mrpg/unpack-ztt to extract readable
treatment definitions before converting.

## Before Writing Any Code

For an existing project, preserve its pinned dependency and deployment choices.
For a new project, install uproot with the supported project scaffold command:

```bash
uv run --with uproot-science uproot setup my_project --minimal
```

If `uv` is unavailable, follow the framework's pip installation guide. Do not
install directly from an unreleased Git branch unless the user explicitly asks.

### Research workflow

1. Read the target project's `main.py`, `pyproject.toml`, and local agent
   instructions when they exist. Preserve their dependency and deployment choices.
2. **Always download the examples repository** if it is not already available:

   ```bash
   git clone https://github.com/mrpg/uproot-examples /tmp/uproot-examples
   ```

   Read its `README.md` and `main.py`, then study two or three relevant complete
   examples: their `__init__.py`, templates, and `simulate.js`. Copy a close
   pattern only when its semantics match.
3. Read `input_elements/__init__.py` before adding form fields. Search examples
   with `rg` for the exact feature instead of guessing; see
   `references/search-patterns.md`.
4. If examples do not settle an API detail, download and search the documentation:

   ```bash
   git clone https://github.com/mrpg/uproot-docs /tmp/uproot-docs
   ```

5. For advanced implementation, lifecycle, or client-API questions that remain
   unclear, download the framework source and inspect the relevant implementation:

   ```bash
   git clone https://github.com/mrpg/uproot /tmp/uproot
   ```

Use existing local clones instead of downloading duplicates. The framework source
is authoritative for signatures and lifecycle behaviour.

Key source files (under `src/uproot/` in the uproot repo):
- `smithereens.py` - Main public helper imports, `@live`, grouping, notifications, page-order operators
- `fields.py` - All field type definitions
- `_static/uproot.js` - Client-side JavaScript API
- `_static/simulate.js` - Simulation helper API (`uproot.simulate`)
- `types.py` - Page base classes and lifecycle enforcement

## App Structure

Every app is a directory containing:

```
my_app/
├── __init__.py      # All Python logic
├── PageName.html    # One template per visible Page class (matching class name exactly)
├── simulate.js      # Automated testing script (optional but recommended)
└── README.md        # Loading instructions
```

Apps are parts of projects. For a new project, use the supported scaffold command
from the documentation, for example `uv run --with uproot-science uproot setup
my_project --minimal`; then work inside that directory. Use `uproot --help` and
`uproot new --help` for the installed version before scaffolding an additional app.
Do not create a project in an existing project's root or overwrite user files.

You must ALWAYS create projects in new subdirectories. NEVER pollute the pwd.

## The __init__.py File

### Required LGPL notice

Every `__init__.py` must preserve the uproot third-party dependency notice.
Because uproot is LGPL v3+, apps that use it must also carry the LGPL license
text for uproot, typically as `uproot_license.txt`. The app itself may be
licensed under 0BSD, MIT, proprietary terms, or any other license compatible
with using an LGPL dependency.

```python
# Docs are available at https://uproot.science/
# Examples are available at https://github.com/mrpg/uproot-examples
#
# This example app is under the 0BSD license. You can use it freely and build on it
# without any limitations and without any attribution. However, these two lines must be
# preserved in any uproot app (the license file is automatically installed in projects):
#
# Third-party dependencies:
# - uproot: LGPL v3+, see ../uproot_license.txt
```

### Standard imports

```python
from uproot.fields import *
from uproot.smithereens import *
```

Add `import uproot.models as um` when using persistent data models (Entry classes).

### Required module-level attributes

```python
DESCRIPTION = "Human-readable description of the experiment"
```

Optional:
```python
SUGGESTED_MULTIPLE = 2      # Hint for session creation (e.g., 2 for pair games)
LANDING_PAGE = True          # Show an app landing page before page_order
```

### Constants class

Use a `C` class for experiment parameters. Access them in templates as `{{ C.NAME }}`:

```python
class C:
    ENDOWMENT = cu("10")    # cu() for currency values
    MULTIPLIER = 3
    ROUNDS = 5
    GROUP_SIZE = 4
```

Export selected constants to JavaScript (`uproot.vars._uproot_internal.C`) by
putting `__export__` inside the `C` class:
```python
class C:
    ITEMS = [...]
    BUCKET_LABELS = [...]
    __export__ = ["ITEMS", "BUCKET_LABELS"]  # or ... to export all C attributes
```

### Page classes

Read `references/page-types.md` for the full reference on page types, hooks, and
lifecycle methods. The essentials:

**Standard page** - displays content, optionally collects form data:
```python
class MyPage(Page):
    fields = dict(
        choice=RadioField(label="Your choice", choices=["A", "B"]),
        amount=DecimalField(label="Amount", min=0, max=100),
    )
```

**NoshowPage** - runs logic without displaying anything:
```python
class Setup(NoshowPage):
    @classmethod
    def after_always_once(page, player):
        player.some_value = compute_something()
```

**GroupCreatingWait** - forms groups from arriving players:
```python
class GroupPlease(GroupCreatingWait):
    group_size = 2

    @classmethod
    def after_grouping(page, group):
        for player, role in zip(group.players, [True, False]):
            player.is_leader = role
```

**SynchronizingWait** - waits for all group members, then runs logic:
```python
class Sync(SynchronizingWait):
    @classmethod
    def all_here(page, group):
        for player in group.players:
            player.payoff = calculate_payoff(player)
```

### Page ordering

```python
page_order = [GroupPlease, Decision, Sync, Results]
```

`page_order` can also be a function for per-player dynamic sequences:
```python
def page_order(player: PlayerType) -> list[Any]:
    if player.treatment == 1:
        return [Instructions, TaskA, Results]
    return [Instructions, TaskB, Results]
```

Advanced constructs (read `references/page-ordering.md`):
- `Rounds(Page1, Page2, n=5)` - repeat a block n times
- `Repeat(Page1, Page2)` - repeat while `player.add_round` is true
- `Between(PageA, PageB, PageC)` - randomly show one
- `Bracket(PageA, PageB)` - group pages together within Between/Random
- `Random(PageA, PageB, PageC)` - randomize order

### Module-level hooks

```python
def new_player(player):
    player.counter = 0
    player.treatment = rng().choice([1, 2])

def new_session(session):
    session.model = um.create_model(session, tag="data")
```

**Note**: `rng` is exported from uproot.smithereens, and returns a random.Random object initialized with
safe cryptographic randomness. In most experiments, `rng()` should be used whenever a random.Random object
could be used, as it is good practice.

Optional app-level callbacks:
- `digest(session)` returns admin-facing summary data (dict or list) for monitoring.
- `pipeline(session)` returns a `list[dict[str, Any]]` of derived rows for export.
- `language(player)` returns the locale string for this player (for i18n).
- `async def restart()` runs on server restart (e.g., to re-create background tasks).
- `async def api2(session, request)` defines an unauthenticated HTTP endpoint.
  Treat its request input as public and untrusted; never expose secrets or
  participant data through it.

## HTML Templates

Read `references/template-patterns.md` for the full reference. The essentials:

Every template extends `Base.html` and defines `title` and `main` blocks:

```html
{% extends "Base.html" %}

{% block title %}
Page Title
{% endblock title %}

{% block main %}

<p>Your endowment is {{ C.ENDOWMENT | fmtnum(places=2) }}.</p>

{{ field(form.amount) }}
{{ errors() }}

{% endblock main %}
```

Key template functions:
- `{{ field(form.fieldname) }}` - render a single form field
- `{{ fields() }}` - render all form fields
- `{{ errors() }}` - display validation errors
- `{{ appstatic("file.js") }}` - URL for static files in the app directory
- `{{ projectstatic("file.js") }}` - URL for project-level static files
- `{% include "app_name/partial.html" %}` - include another template
- `{{ chat(session.chat) }}` - render a chat widget

Key template variables:
- `{{ C.CONSTANT }}` - constants from the C class
- `{{ player.attribute }}` - player data
- `{{ player.payoff }}` - player's payoff
- `{{ player.context.property }}` - computed context properties
- `{{ player.other_in_group.attribute }}` - partner's data (2-player groups)
- `{{ player.others_in_group }}` - list of other group members
- `{{ player.group.players }}` - all group members

### The `| safe` filter

When a template variable contains HTML that should be rendered as markup (not
escaped), use `{{ variable | safe }}`. This is needed for experimenter-defined
content such as instructions stored in constants, formatted prompts, or
dynamically built HTML from Python code. **Only use `| safe` on values that
come from the experimenter's code (e.g., `C.INSTRUCTIONS`, computed context
properties, Python-generated HTML). Never use `| safe` on any value that could
contain participant input** — player-typed text must always be auto-escaped by
omitting the filter.

Use `{% block head %}` for custom CSS and `{% block late %}` for late-loaded scripts.

Additional blocks: `pre_container`, `main_full_width` (no container),
`main2`/`main2_full_width`/`main3` (extra content sections),
`pre_main`/`post_main`, `pre_form`/`form_start`/`form_end`,
`header_start`/`header_end`, `footer`, `late2`.

Template switches (set as Jinja2 variables):
- `buttons = false` - hide the default Next button
- `disable_bootstrap = true` - omit Bootstrap CSS/JS
- `disable_uproot_fonts = true` - omit default web fonts
- `disable_tabular_numbers = true` - omit the tabular-number font stylesheet
- `disable_terms = true` - omit the terms script
- `disable_auto_start = true` - don't auto-initialize uproot JS
- `disable_connection_lost_modal = true` - suppress connection-lost modal

Bootstrap 5 and Alpine.js are available on every page out of the box.

### UX and Accessibility

Economic experiments depend on participants understanding instructions and
interacting with the interface without confusion. Follow these principles:

- **Use Bootstrap components properly.** Use `form-label`, `form-control`,
  `form-check`, `btn`, `card`, `alert`, and grid classes (`row`, `col-*`)
  as intended. Don't reinvent layout with custom CSS when Bootstrap provides it.
- **Accessible forms.** Every input must have an associated `<label>` (uproot's
  `{{ field() }}` handles this). For custom inputs, use `<label for="id">` or
  `aria-label`. Group related radio buttons with `<fieldset>` and `<legend>`.
- **Clear, concise instructions.** Write short sentences. Bold key terms or
  amounts (`<strong>{{ C.ENDOWMENT }}</strong>`). Use lists for multi-step
  instructions. Avoid jargon and academic language in participant-facing text.
- **Visual hierarchy.** Use headings (`<h4>`, `<h5>`) to structure content.
  Separate distinct sections with cards or spacing (`mb-3`, `mt-4`). Put the
  primary action (the submit button) in a visually prominent position.
- **Sufficient contrast and font size.** Stick with Bootstrap's default
  typography. Don't use light gray text or small font sizes for important
  content. Use `alert-info`, `alert-warning`, etc. for callouts.
- **Responsive layout.** Use Bootstrap's grid so the experiment works on
  different screen sizes. If in doubt, follow best practices.

## simulate.js

Every app should include a `simulate.js` for automated testing. It runs on player
pages in sessions created with "Simulate responses" enabled. Use the
`uproot.simulate` API — it provides chainable helpers for filling fields,
choosing radio buttons, and submitting:

```javascript
uproot.simulate.on("my_app/Decision", (sim) => {
    sim.fill("amount", sim.integer(0, 100)).submit();
});
```

For radio buttons:
```javascript
uproot.simulate.on("my_app/Choice", (sim) => {
    sim.choose("choice", sim.random(["A", "B", "C"])).submit();
});
```

For multiple fields:
```javascript
uproot.simulate.on("my_app/Survey", (sim) => {
    sim.fill({
        response: sim.random(["red", "green", "blue"]),
        reaction_time_ms: String(sim.integer(250, 1200)),
    }).submit();
});
```

Available `sim` methods:
- `sim.fill(name, value)` or `sim.fill({name: value, ...})` - set input values
- `sim.choose(name, value)` - select a radio button or dropdown option
- `sim.check(name)` / `sim.uncheck(name)` - toggle checkboxes
- `sim.chooseAnyRadio()` - pick a random radio button on the page
- `sim.oneOf(name, values)` - choose a random value from an array
- `sim.random(array)` - return a random element from an array
- `sim.integer(min, max)` - return a random integer in [min, max]
- `sim.submit()` - submit the page
- All methods except `random`/`integer` return `sim` for chaining

Pages without form fields (e.g., results pages) can auto-advance:
```javascript
uproot.simulate.on("my_app/Results", (sim) => {
    sim.submit();
});
```

## Registration

Add the app to `main.py`:
```python
load_config(uproot_server, config="my_app", apps=["my_app"])
```

Add to the Apps table in `README.md` with description and difficulty rating.

## Field Types

Read `references/field-types.md` for the complete reference. Summary of available
types: BICField, BooleanField, BoundedChoiceField, DateField, DecimalField,
DecimalRangeField, EmailField, FileField, IBANField, IntegerField, LikertField,
RadioField, SelectField, StringField, TextAreaField.

## Live Methods (WebSocket)

For real-time interaction without page reloads. Live methods can be sync or async:

```python
class MyPage(Page):
    @live
    def do_something(page, player, value: int):
        player.data = value
        return {"status": "ok", "new_value": player.data}
```

Call from JavaScript:
```javascript
uproot.invoke("do_something", 42).then(result => {
    document.getElementById("output").textContent = result.new_value;
});
```

Use `may_proceed()` to control when the player can advance past a live page.

## Persistent Data Models

For complex data that doesn't fit on a player (e.g., per-trial measurements,
order books, transaction logs):

```python
import uproot.models as um

class Trial(metaclass=um.Entry):
    pid: PlayerIdentifier
    trial_number: int
    response: str
    correct: bool
    reaction_time_ms: float

def new_session(session):
    session.trials = um.create_model(session, tag="trials")
```

Store entries: `um.add_entry(session.trials, player, Trial, trial_number=1, ...)`
Query entries: `um.filter_entries(session.trials, Trial, pid=player.pid)`

## Context Classes

For computed properties accessible in templates as `player.context.property`:

```python
class Context(PlayerContext):
    @property
    def total(self):
        return sum(p.contribution for p in self.player.group.players)
```

## Chat Integration

```python
def new_session(session):
    session.chat = chat.create(session)

class ChatPage(Page):
    @classmethod
    def before_once(page, player):
        chat.add_player(player.session.chat, player, pseudonym=f"Player {player.id}")
```

Template: `{{ chat(session.chat) }}`

## Checklist for New Apps

1. Study the relevant examples (see Research workflow above) and adapt a similar existing app where appropriate
2. Preserve the required uproot LGPL notice and include `uproot_license.txt`
3. Define `DESCRIPTION` and `page_order`
4. Create matching `.html` templates for each Page class (name must match exactly)
5. Add to `main.py` with `load_config()`
6. Add to `README.md` in the Apps table
7. Write a `simulate.js`
8. Run `black`, `isort`, `ruff` if installed (star import warnings may be ignored)
9. Use 4-space indentation everywhere

## Common Paradigms

For game theory experiments (dictator, trust, ultimatum, public goods, prisoner's
dilemma, Cournot, Bertrand, Stackelberg, etc.), read `references/game-theory.md`.

For real effort tasks, surveys, quizzes, and other specialized patterns, grep the
examples repository for similar apps before building from scratch.

## Testing

```bash
uv run uproot run    # or: uproot run
```

After building or modifying an app, tell the user clearly how to correctly run
the uproot server, and how to access the admin area from the browser.

**CRITICAL: Server lifecycle.** If you start the uproot server (e.g., to verify
the app loads), you **must** kill it before returning control to the user. Never
leave the server running in the background — it occupies the port and blocks
future runs. Always run the server with a timeout (e.g., `timeout 15 uv run
uproot run`) and confirm the process has terminated. If you used
`run_in_background`, stop the process explicitly before finishing. The user will
start the server themselves when they are ready to test.

**Do NOT** attempt to log in via curl, access player pages programmatically, test
WebSocket connections, or interact with the running app from the command line
unless the user specifically asks you to. The browser is the intended test
interface; automated HTTP/WebSocket probing is fragile and unnecessary for normal
development.

## Handling Uncertainty

If the AskUserQuestion tool is available, use it to ask the user whenever you
encounter:
- Uncertainties about requirements or intended behavior
- Important missing details (e.g., number of players, payoff rules, round count)
- Contradictions between what the user asked and what the code/examples show
- Important design choices where multiple valid approaches exist

Do not guess or assume — ask. A brief clarifying question saves far more time
than building the wrong thing.
