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

## Jinja2 Essentials

### Variables
```html
{{ player.name }}
{{ player.payoff }}
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
{{ value | to(1) }}                     {# Format to 1 decimal place #}
{{ data | tojson }}                     {# JSON encode for JavaScript #}
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
{{ appstatic("script.js") }}            {# URL for static file in app dir #}
{% include "app_name/partial.html" %}   {# Include another template #}
```

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

Alpine.js is available on every page. Use it for reactive UIs:

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

### Bootstrap 5

Bootstrap 5 CSS and JS are available on every page. Use Bootstrap classes for
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

## Template Includes

Put repeating parts in separate HTML files:

```html
{% include "my_app/PayoffTable.html" %}
```

The included file does NOT use `{% extends %}` - it's a fragment:

```html
{# PayoffTable.html - no extends block #}
<table class="table">
    <tr>
        <td>Your payoff</td>
        <td>{{ player.payoff }}</td>
    </tr>
</table>
```
