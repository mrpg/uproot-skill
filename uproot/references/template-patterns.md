# Template and JavaScript Patterns

## HTML Template Structure

Every page template is a `.html` file named exactly after the page class. It
extends `Base.html` and defines blocks:

```html
{% extends "Base.html" %}

{% block title %}
Page Title
{% endblock title %}

{% block main %}
<!-- Page content here -->
{% endblock main %}
```

### Additional blocks

```html
{% block head %}
<style>
    .custom-class { color: red; }
</style>
{% endblock head %}

{% block late %}
<script>
    // Late-loaded JavaScript (after page content)
</script>
{% endblock late %}
```

| Block | Position |
|---|---|
| `title` | Browser tab and page heading |
| `head` | Inside `<head>` |
| `pre_main` | Before the main content, outside the Bootstrap container |
| `main` | Main content, inside the container |
| `main_full_width` | After `main`, full viewport width (banners, wide charts) |
| `main2` | A second container section |
| `late` | End of `<body>` (scripts) |
| `timeoutbox` | Countdown display; override empty to hide it |

Further slots: `main2_full_width`, `main3`, `late2`, `footer`, `header_start`,
`header_end`. There is no `pre_container` block.

### Markdown pages

If `PageName.html` is missing, uproot renders `PageName.md`. It needs exactly one
level-one heading, which becomes the title. Jinja works as usual:

```markdown
# Welcome

You start with {{ C.ENDOWMENT }} points.

{{ field(form.consent) }}
```

Use Markdown for text-heavy pages (instructions, consent, debriefing); use HTML
when the page needs custom layout or scripts. See the `anchoring_markdown` example.

### Base template switches

Set these near the top of a page template to change the default layout:

```html
{% extends "Base.html" %}
{% set buttons = False %}
```

| Variable | Default | Effect |
|---|---|---|
| `buttons` | `True` | `False` hides the default Back/Next buttons |
| `disable_alpinejs` | `False` | Do not load Alpine.js |
| `disable_bootstrap` | `False` | Do not load Bootstrap CSS/JS |
| `disable_uproot_fonts` | `False` | Do not load uproot's web fonts |
| `disable_tabular_numbers` | `False` | Do not load the tabular-number font stylesheet |
| `disable_terms` | `False` | Do not load the terms script |
| `disable_auto_start` | `False` | Do not run `uproot.init()` and the WebSocket start |
| `disable_connection_lost_modal` | `False` | No connection-lost modal |

To change a default for the whole project, set it once in `main.py` instead of
in every template (DRY):

```python
upd.TEMPLATE_DEFAULTS["disable_alpinejs"] = True
```

`TEMPLATE_DEFAULTS` also accepts `buddy_image` (a file found by `static()`) and
`buddy_label` to customize the chat buddy.

## Compose Before You Duplicate

Apply the mandatory DRY pass from `SKILL.md` before writing page-local frontend
code. Reuse is based on ownership and scope: global project composition, scoped
static assets, and parameterised Jinja2 fragments. These mechanisms work for
HTML, CSS, JavaScript, and arbitrarily complex Jinja2.

### ProjectHead.html and ProjectBody.html

uproot automatically includes root-level `ProjectHead.html` inside `<head>` and
root-level `ProjectBody.html` at the end of `<body>` on participant-facing pages.
They are fragments: do not extend `Base.html` or wrap their content in page blocks.

Use them for concerns that truly apply throughout the project, such as a shared
stylesheet, shared browser behaviour, a global progress indicator, or study-wide
chrome. Prefer links to project static assets over copying `<style>` or `<script>`
contents into every page:

```text
project/
├── ProjectHead.html
├── ProjectBody.html
├── _static/
│   ├── study.css
│   └── study.js
└── my_app/
    ├── _static/
    │   └── decision.js
    └── ChoiceCard.html
```

```html
{# ProjectHead.html #}
<link rel="stylesheet" href="{{ projectstatic('study.css') }}">
<script defer src="{{ projectstatic('study.js') }}"></script>
```

Project fragments can render on routes without a current participant. Guard
participant-dependent markup explicitly:

```html
{# ProjectBody.html #}
{% if player is not none and len(player.page_order) > 0 %}
    <div class="study-progress" data-page="{{ player.show_page }}"></div>
{% endif %}
```

Do not put app-specific assumptions into these global fragments. In particular,
use `projectstatic()` there; `appstatic()` requires a current app context.

### Static assets by scope

Place project-wide files in the root `_static/` directory and address them with
`projectstatic()`. Place app-owned files in `<app>/_static/` and address them with
`appstatic()`. Both helpers accept nested paths. `static('file')` searches the
current app's `_static/` first and falls back to the project's, which suits
defaults that individual apps may override.

```html
{% block head %}
<link rel="stylesheet" href="{{ appstatic('css/task.css') }}">
{% endblock head %}

{% block late %}
<script src="{{ appstatic('js/task.js') }}"></script>
{% endblock late %}
```

Never hard-code `/static/...` URLs. The helpers preserve uproot's configured root
and correctly encode path components. Keep pure reusable CSS and JavaScript in
static files. Inline only genuinely page-specific snippets or Jinja2-generated
configuration.

### Parameterised template fragments

An included fragment inherits the template context. Use `{% with %}` to give it a
small, explicit interface, and use loops to render repeated structures from data:

```html
{% for option in options %}
    {% with field_name="choice_" ~ loop.index, option=option %}
        {% include "my_app/ChoiceCard.html" %}
    {% endwith %}
{% endfor %}
```

```html
{# my_app/ChoiceCard.html: a fragment, with no extends or page blocks #}
<div class="card">
    <label for="{{ field_name }}">{{ option.label }}</label>
    <input id="{{ field_name }}" name="{{ field_name }}" value="{{ option.value }}">
</div>
```

Fragments may contain markup, `<style>`, `<script>`, loops, conditionals, nested
includes, and other Jinja2. Use a static file when the CSS or JavaScript is pure;
use a fragment when it needs server-rendered values or Jinja2 control flow. A
required `PageName.html` may be a thin wrapper that supplies variables and includes
one shared implementation. Pages or treatments that differ only in data must not
carry copied template bodies.

For substantial browser behaviour, keep the implementation in one static file and
pass page data through the page's `jsvars`, accessed as `uproot.vars`:

```python
class Task(Page):
    @classmethod
    def jsvars(page, player):
        return dict(limit=C.LIMIT, treatment=player.treatment)
```

```html
{% block late %}
<script src="{{ appstatic('task.js') }}"></script>
{% endblock late %}
```

This keeps templates declarative and prevents multiple inline copies of the same
event handling. Extract ordinary JavaScript helper functions inside `simulate.js`
when several simulated pages share fill or submission behaviour.

## Jinja2 Essentials

### Variables
```html
{{ player.name }}
{{ player.payoff | fmtnum(places=2) }}
{{ player.get("field_name") }}          {# Safe access, returns None if missing #}
{{ C.ENDOWMENT }}
{{ app.LABELS.key }}
{{ player.context.computed_property }}
```

### Conditionals
```html
{% if player.is_leader %}
    <p>You are the leader.</p>
{% else %}
    <p>You are the follower.</p>
{% endif %}

{% if value is not none %}...{% endif %}
```

### Loops
```html
{% for p in player.others_in_group %}
    <li>Player {{ p.id }}: {{ p.contribution }}</li>
{% endfor %}

{% for round in range(1, C.ROUNDS + 1) %}
    <tr>
        <td>{{ round }}</td>
        <td>{{ player.within(round=round).get("choice") }}</td>
    </tr>
{% endfor %}
```

### Inline expressions
```html
{{ "Yes" if player.cooperated else "No" }}
{{ value | fmtnum(places=1) }}          {# Format to 1 decimal place #}
{{ data | tojson }}                     {# JSON encode; no `| safe` needed #}
{{ C.INSTRUCTIONS | safe }}             {# Render experimenter HTML as markup #}
```

### The `| safe` filter

Use `| safe` when the value contains HTML that should be rendered, not escaped.
This is appropriate for experimenter-defined content: constants with HTML markup,
context properties that build formatted strings, or Python-generated HTML.

```html
{{ C.TASK_PROMPT | safe }}              {# Constant with <b>, <ul>, etc. #}
{{ player.context.summary | safe }}     {# Computed HTML from Python #}
```

**Never use `| safe` on participant-typed values.** Any data that originates
from player input (form fields, chat messages, free-text responses) must remain
auto-escaped to prevent XSS. If in doubt, omit the filter.

## Template Functions

```html
{{ field(form.fieldname) }}             {# Render a form field #}
{{ fields() }}                          {# Render all fields #}
{{ errors() }}                          {# Display form errors #}
{{ chat(session.chat) }}                {# Render chat widget #}
```

See **Compose Before You Duplicate** for `ProjectHead.html`, `ProjectBody.html`,
`appstatic()`, `projectstatic()`, and `{% include %}`.

## Player Data Access

### Current player
```html
{{ player.attribute }}
{{ player.payoff }}
{{ player.round }}                      {# Current round number #}
{{ player.treatment }}
```

### Group access
```html
{{ player.group.name }}
{{ player.group.players }}              {# All group members #}
{{ player.other_in_group.attribute }}   {# Partner in 2-player group #}
{{ player.others_in_group }}            {# Other members (excludes self) #}
```

### Historical data (rounds)
```html
{{ player.within(round=1).get("choice") }}
{{ player.within(app="my_app").along("round") }}  {# Iterate round history #}
```

## JavaScript Integration

### Built-in client API

```javascript
// Submit the current page form
uproot.submit();

// Call a @live method on the current page
uproot.invoke("method_name", arg1, arg2).then(result => {
    // handle result
});

// Get the current page identifier
uproot.currentPage  // e.g., "my_app/Decision"

// Access jsvars data
uproot.vars.myVariable

// Constants listed in C.__export__
C.MY_CONSTANT

// Format numbers exactly like the fmtnum filter (rounds half up)
uproot.fmtnum(1234.5, { pre: "$", places: 2 })  // "$1,234.50"

// Call the app's api2() as the authenticated participant (returns a fetch Promise)
uproot.api2("my_app", { method: "POST", body: JSON.stringify(payload),
                        headers: { "Content-Type": "application/json" } })

// Run code when page is ready
uproot.onReady(() => {
    // initialization
});

// Receive server-sent notifications
uproot.receive = (data) => {
    // handle notification from notify()
};

// Listen for notify(..., event="Notified")
uproot.onCustomEvent("Notified", (event) => {
    // event.detail.data contains the payload
});
```

### Form field access in JavaScript

```javascript
I("fieldname")                          // Get input element by field name
I("fieldname").value = 42;              // Set value
I("radio-0").checked = true;            // Check a radio button
```

### Alpine.js

Alpine.js is loaded on every page unless `disable_alpinejs` is set. Use it for
reactive UIs:

```html
<div x-data="{ count: 0, loading: false }">
    <p x-text="count"></p>
    <button @click="count++">+1</button>
    <div x-show="loading">Loading...</div>
</div>
```

Common Alpine.js patterns in uproot:
```html
<div x-data="{ state: 'ready' }" x-init="init()">
    <template x-if="state === 'ready'">
        <button @click="startTrial()">Begin</button>
    </template>
    <template x-if="state === 'trial'">
        <div :style="{ color: trialColor }">
            <span x-text="trialWord"></span>
        </div>
    </template>
</div>
```

The built-in Alpine store exposes the page timeout:
`Alpine.store("uproot").timeout` (`{ active, level, text, compact }`).

### Bootstrap 5

Bootstrap 5 CSS and JS (and Bootstrap Icons) are available on every page. The
`bootstrap_elements` example is a guided tour of what is available. Use Bootstrap classes for
layout, forms, buttons, cards, modals, alerts, and responsive design:

```html
<div class="row">
    <div class="col-md-6">
        <div class="card">
            <div class="card-body">
                {{ field(form.amount) }}
            </div>
        </div>
    </div>
</div>

<button type="button" class="btn btn-primary" onclick="doSomething()">
    Click me
</button>

<div class="alert alert-info">
    Important information here.
</div>
```
