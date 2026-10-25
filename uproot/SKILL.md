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
  (DecimalField, FloatField, RadioField, LikertField, etc.), uproot imports
  (uproot.smithereens, uproot.fields, uproot.models), page-order operators
  (Rounds, Repeat, RandomOrder, Between, Bracket, blocks), or asks about rounds,
  groups, live/WebSocket features, or simulate.js in an experiment context.
---

# Building Uproot Experiments

uproot is a Python framework for browser-based behavioral experiments. Apps are
self-contained directories with Python logic, Jinja2 HTML templates, and optional
JavaScript. This skill guides you through building them correctly.

## CRITICAL: DON'T REPEAT YOURSELF

**DRY is a non-negotiable design invariant. Every fact, rule, fragment, style, and
behaviour must have one authoritative implementation. Respect this throughout every
task, not merely during cleanup.** This applies equally to Python, HTML, Jinja2,
JavaScript, CSS, `simulate.js`, study copy, constants, and payoff logic.

Before adding or changing code:

1. Search the project for the same content, selector, calculation, field pattern,
   event handler, and component. Inspect `ProjectHead.html`, `ProjectBody.html`,
   project/app `_static/` directories, and existing template fragments.
2. Identify the narrowest correct owner: project, app, page family, or page.
3. Extend that owner and update its callers. Do not paste a second implementation
   and do not leave the superseded copy behind.

After editing, search again for stale or parallel implementations. A change is not
complete while the same knowledge must be maintained in multiple places. Prefer a
small function, data structure plus loop, context property, Jinja2 fragment, or
static asset over copied code. Keep abstractions surgical: consolidate genuinely
shared knowledge, but do not build machinery around an unrelated one-off.

For frontend work, use uproot's excellent reuse facilities as the default architecture:

| Scope | One authoritative home | Consume it with |
|---|---|---|
| Every participant page in the project | root `ProjectHead.html` or `ProjectBody.html` | automatic inclusion by uproot |
| Shared project CSS, JavaScript, images, or documents | root `_static/` | `projectstatic()` |
| One app's CSS, JavaScript, images, or documents | `<app>/_static/` | `appstatic()` |
| Repeated or parameterised markup/Jinja2 | an app template fragment | `{% include %}`, usually with `{% with %}` |

Use Python helpers and `C` for repeated definitions and parameters. Keep authoritative
experimental calculations on the server; do not independently reimplement them in
templates and JavaScript. When interactive previews require client-side calculation,
put that calculation in one JavaScript helper and verify it against the server rule.
For the occasional experiment whose same complicated formula genuinely must run in
both Python and JavaScript, consider [exprcompile](https://github.com/mrpg/exprcompile).
It compiles one SymPy expression to both languages, including complete source files
via `compile_all_to_files`, so the symbolic formula remains a single source of truth.
Do not add it for simple or server-only calculations.
Read `references/template-patterns.md` for the exact composition patterns.

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
- `deployment.py` - Settings for `main.py` (`upd.*`), including `TEMPLATE_DEFAULTS`
- `models.py` - Persistent data models (`uproot.models`)

The uproot-docs repository has an `upgrade.md` page, and uproot announces breaking
changes and security notices via `uv run uproot announcements`. When an existing
project pins an older uproot, check its installed version (`uv run uproot --version`)
before applying APIs from this skill that may be newer than the pin.

## App Structure

Every app is a directory containing:

```
my_app/
├── __init__.py          # All Python logic
├── PageName.html        # One template per visible Page class (or PageName.md)
├── _static/             # App-scoped CSS, JavaScript, images, and documents
├── simulate.js          # Automated testing script (optional but recommended)
├── AdminDigest.html     # Optional: admin view of digest(session)
├── AdminSettings.html   # Optional: custom session-settings form in the admin
└── README.md            # Loading instructions
```

Templates are looked up by class name: first `PageName.html`, then `PageName.md`
(a Markdown page with exactly one `#` heading, which becomes the title; Jinja
still works). An explicit `template = f"{__name__}/Other.html"` is relative to
the **project root**, so always include the app name.

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
text for uproot, typically as `uproot_license.txt` (new projects get it
automatically). The app itself may be licensed under 0BSD, MIT, proprietary
terms, or any other license compatible with using an LGPL dependency. This is
the header that `uproot new --minimal` writes:

```python
# SPDX-License-Identifier: 0BSD
#
# Third-party dependencies:
# - uproot: LGPL v3+, see ../uproot_license.txt
#
# Docs are available at https://uproot.science/
# Examples are available at https://github.com/mrpg/uproot-examples
```

When adapting an example app, keep its header, including the two
"Third-party dependencies" lines.

### Standard imports

```python
from uproot.fields import *
from uproot.smithereens import *
```

Add `import uproot.models as um` when using persistent data models (Entry classes).

New projects run `mypy --strict`, and the examples annotate every hook. Follow
suit: annotate `player: PlayerType`, `group: GroupType`, `session: SessionType`
(all exported from `uproot.smithereens`) and return types (`-> None`,
`-> dict[str, Any]`, …). Use `cast()` where a stored value's type is unknown to
mypy, e.g. `return cast(bool, player.done)`.

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

Export selected constants to JavaScript (available as `window.C`, i.e. `C.ITEMS`
in scripts and Alpine expressions) by putting `__export__` inside the `C` class:
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

Advanced constructs ("SmoothOperators"; read `references/page-ordering.md`):
- `Rounds(Page1, Page2, n=5)` - repeat a block n times (`n` must be an `int`)
- `Repeat(Page1, Page2)` - repeat while `player.add_round` is true
- `Between(PageA, PageB, PageC)` - randomly show one
- `Bracket(PageA, PageB)` - group pages together within Between/RandomOrder
- `RandomOrder(PageA, PageB, PageC)` - randomize order (formerly `Random`, which no longer exists)
- `block="main"` on `Rounds`/`Repeat` - **required** when one app can run several loops

Operators accept only pages and other operators. Never pass a list; use `Bracket()`.

### Module-level hooks

```python
def new_player(player: PlayerType) -> None:
    player.counter = 0
    player.treatment = rng().choice([1, 2])

def new_session(session: SessionType) -> None:
    session.model = um.create_model(session, tag="data")
```

**Note**: `rng` is exported from uproot.smithereens, and returns an independent
`random.Random` object seeded from the OS. Always use `rng()` instead of the
module-level `random` functions. If later draws must continue the same stream,
store the generator itself on the player (`player.rng = rng()`); its state is
persisted.

**Reserved names.** Never use these as field or attribute names: `name` (the
generated player name; use e.g. `participant_name`), `along`, `fields`, `flush`,
`get`, `history`, `refresh`, `strict`, `within`. uproot also manages `round`,
`block`, `round_nested`, `add_round`, and `between_showed`.

Optional app-level callbacks (they are called with keyword arguments, so use
exactly these parameter names):
- `digest(session)` returns a dict for the admin's digest view, rendered by
  `AdminDigest.html` (the dict's keys become template variables).
- `pipeline(session, data=None)` returns a `list[dict[str, Any]]` of derived rows
  that the admin can run and download. `data` is posted JSON or `None`.
- `language(player)` returns the locale string for this player (for i18n).
- `async def restart()` runs on server restart (e.g., to re-create background tasks).
- `api(request, session)` is an operator endpoint that requires an API key.
- `api2(request, session, player=None)` is a participant endpoint at
  `/api2/{app}/{session}/`. It is **public** unless called from the participant's
  page via `uproot.api2(appname, fetchOptions)`, in which case `player` is the
  authenticated participant. Reject `player is None` when you need a participant;
  never expose secrets, admin data, or other participants' data through it.

Read `references/advanced.md` for these callbacks and for `AdminSettings.html`.

## HTML Templates

Read `references/template-patterns.md` whenever creating or changing HTML, Jinja2,
CSS, or browser JavaScript. It is the single reference for template structure,
safe rendering, blocks, client APIs, and the DRY composition facilities highlighted
above. Bootstrap 5 and Alpine.js are available on every page out of the box
(both can be switched off per template or project-wide, see that reference).

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
  content. Use `alert-info`, `alert-warning`, `callout`, etc. for callouts.
- **Typographical quotes and apostrophes in body text.** In regular text and in
  headings, use typographical quotes (U+201C “…” U+201D) instead of straight quotes ("..."),
  and use the typographical apostrophe (U+2019 ’) instead of the straight apostrophe
  (').
- **No spaces around em dashes.** Do not surround em dashes (U+2014 —) by spaces.
- **Responsive layout.** Use Bootstrap's grid so the experiment works on
  different screen sizes. If in doubt, follow best practices.

## simulate.js

Every app should include a `simulate.js` for automated testing. It lives in the
app directory (there is no project-wide `simulate.js`) and runs on player pages
in sessions created with "Simulate responses" enabled, or started with
`uv run uproot start <config> --simulate`. Use the
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

`uproot.simulate.on(expected, fn)` matches the page identifier `"app/Page"`;
`expected` may also be an array, a `RegExp`, or a predicate function.

Available `sim` methods:
- `sim.fill(name, value)` or `sim.fill({name: value, ...})` - set input values
- `sim.choose(name, value)` / `sim.select(name, value)` - select a radio button or dropdown option
- `sim.check(name)` / `sim.uncheck(name)` - toggle checkboxes
- `sim.chooseAnyRadio()` - pick a random radio button on the page
- `sim.oneOf(name, values)` - choose a random value from an array
- `sim.value(name)` - read a field's current (or checked radio) value
- `sim.assertPage(expected)` - throw if not on the expected page
- `sim.random(array)` - return a random element from an array
- `sim.integer(min, max)` - return a random integer in [min, max]
- `sim.submit()` - submit the page
- All methods except `random`/`integer`/`value` return `sim` for chaining

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

uproot also registers every app on its own as a config named `~my_app`. Such
single-app configs have **no settings**, so read settings with defaults
(`get_setting(session, "n_rounds", 5)`) or fail clearly in `new_session`.

## Field Types

Read `references/field-types.md` for the complete reference. Summary of available
types: BICField, BooleanField, BoundedChoiceField, DateField, DecimalField,
DecimalRangeField, EmailField, FileField, FloatField, FloatRangeField, IBANField,
IntegerField, LikertField, LikertFieldClassic, RadioField, SelectField,
StringField, TextAreaField. Nested WTForms fields (`FormField`, `FieldList`) are
rejected.

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
(pass the player itself; identifier fields are filled automatically)
Query entries: `um.filter_entries(session.trials, Trial, pid=player.pid)`

## Context Classes

For computed properties accessible in templates as `player.context.property`:

```python
class Context(PlayerContext):
    @property
    def total(self):
        return sum(p.contribution for p in self.player.group.players)
```

`player.context` follows the participant's *current* app, so use it only in page
methods and templates. In `page_order(player)`, `new_player`, `digest`,
`pipeline`, and other app-level functions, write `Context(player)` instead.

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
2. Complete the mandatory DRY search-and-ownership pass
3. Preserve the required uproot LGPL notice and include `uproot_license.txt`
4. Define `DESCRIPTION` and `page_order`
5. Create matching `.html` templates for each Page class (name must match exactly)
6. Add to `main.py` with `load_config()`
7. Add to `README.md` in the Apps table
8. Write a `simulate.js`
9. Run `black`, `isort`, `ruff`, and `mypy` (strict, as configured in new projects), and fix issues rigorously and cleanly
10. Use 4-space indentation everywhere
11. If the project has translation YAML files, run `uv run uproot check-translations`

## Common Paradigms

For game theory experiments (dictator, trust, ultimatum, public goods, prisoner's
dilemma, Cournot, Bertrand, Stackelberg, etc.), read `references/game-theory.md`.

For real effort tasks, surveys, quizzes, and other specialized patterns, grep the
examples repository for similar apps before building from scratch.

## Testing

```bash
uv run uproot run                       # normal server; admin login link is printed
uv run uproot start my_config           # also creates an open room and prints its URL
uv run uproot start my_config --simulate  # same, with simulate.js enabled
```

`uproot start` is the fastest way to try a config: open the printed room URL in
one browser tab per participant. The config argument may be omitted if the
project has exactly one config.

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
