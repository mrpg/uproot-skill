# Field Types Reference

All field types are imported via `from uproot.fields import *`.

Read `input_elements/__init__.py` in the examples repository for a live
demonstration of every field type with all their parameters.

## Available Field Types

### StringField
Single-line text input.
```python
StringField(label="Your name")
StringField(label="Code", render_kw={"placeholder": "Enter code"})
```

### TextAreaField
Multi-line text input.
```python
TextAreaField(label="Comments", render_kw={"rows": 5})
```

### IntegerField
Whole number input.
```python
IntegerField(label="Quantity", min=0, max=100)
```

### DecimalField
Decimal number input. Use this for money.
```python
DecimalField(label="Amount", min=0, max=cu("10"), places=2)
DecimalField(label="Price", min=0, max=C.MAX_PRICE, step="0.01")
```

### FloatField
Like `DecimalField` (same options), but stores a Python `float`. Use it for
measurements such as reaction times, not for money.
```python
FloatField(label="Reaction time (seconds)", min=0.0, step=0.001)
```

### DecimalRangeField
Slider input for decimal values.
```python
DecimalRangeField(
    label="How much do you agree?",
    min=0, max=100, step=1,
    label_min="Disagree", label_max="Agree",
)
```

Long endpoint labels wrap below `breakpoint` (viewport width in px, default 992).

### FloatRangeField
Like `DecimalRangeField` (same slider options), but stores a `float`.

### BooleanField
Checkbox input.
```python
BooleanField(label="I agree to the terms")
```

### RadioField
Radio button group for single selection.
```python
RadioField(
    label="Your choice",
    choices=["Option A", "Option B", "Option C"],
)
RadioField(
    label="Cooperate?",
    choices=[(True, "Cooperate"), (False, "Defect")],
)
```

### SelectField
Dropdown select.
```python
SelectField(
    label="Country",
    choices=["Germany", "France", "UK"],
)
```

The dropdown starts with an empty option ("—"), so skipping the question cannot
submit the first choice by accident. Required fields reject it; optional fields
save `None`. If your choices already contain `""` (e.g. `("", "Please choose")`),
no extra option is added and an optional field saves `""`.

### LikertField
Likert scale (radio buttons in a row; responsive below `breakpoint`, default 992px).
```python
LikertField(
    label="I enjoy this task",
    min=1, max=5,
    label_min="Strongly disagree",
    label_max="Strongly agree",
)
```

`choices=` overrides the auto-generated integer choices. `LikertFieldClassic`
takes the same options except `breakpoint` and always renders as a table.

For several items sharing one scale, render a matrix in the template instead of
one field per row. These macros do not show errors, so add `{{ errors() }}`:
```html
{{ likert_matrix(form.item1, form.item2, form.item3) }}  {# table #}
{{ likert_grid(form.item1, form.item2, breakpoint=992) }}  {# responsive #}
{{ errors() }}
```

### DateField
Date picker input.
```python
DateField(label="Date of birth")
```

### EmailField
Email input. It uses the browser's email input type; the server-side
`wtforms.validators.Email()` validator is not enabled by default in uproot.
```python
EmailField(label="Your email")
```

### FileField
File upload input.
```python
FileField(label="Upload your CV")
```

### IBANField
IBAN input with validation. `IBANField` and `BICField` require the optional extra:
`uv add 'uproot-science[iban]<1'`.
```python
IBANField(label="Your IBAN")
```

### BICField
BIC/SWIFT code input.
```python
BICField(label="Your BIC")
```

### BoundedChoiceField
Choice field with configurable bounds on how many options can be selected.
```python
BoundedChoiceField(
    label="Select 2-3 options",
    choices=["A", "B", "C", "D", "E"],
    min=2, max=3,
)
```

## Common Parameters

These parameters are available on many uproot field wrappers. Check the field
constructor in `src/uproot/fields.py` when combining less common options:

```python
SomeField(
    label="Display label",
    description="Help text shown below the field",
    optional=True,               # Most input fields except Boolean/BoundedChoice
    addon_start="$",             # String/TextArea/Integer/Decimal/IBAN
    addon_end="per unit",        # String/TextArea/Integer/Decimal/IBAN
    class_addon_start="...",     # Addon CSS class where addons are supported
    class_addon_end="...",       # Addon CSS class where addons are supported
    class_wrapper="col-md-6",    # CSS wrapper class (most wrappers)
    label_floating="Short label",# Floating label text (String/TextArea/Email/IBAN)
    render_kw={"placeholder": "hint", "class": "extra-class"},
)
```

For RadioField/BoundedChoiceField:
```python
RadioField(
    choices=[("a", "A"), ("b", "B")],
    layout="horizontal",  # Adds Bootstrap inline radio classes
)
```

uproot's choice fields accept tuples, dicts, or plain value lists. Raw
`wtforms.*` choice fields (rarely needed) require `wtforms.SelectChoice(value,
label)` instead of tuples. Nested `wtforms.FormField`/`FieldList` are rejected.

For DecimalRangeField:
```python
DecimalRangeField(
    min=0, max=10,
    anchoring=False,     # Disable slider anchoring behavior
    hide_popover=True,   # Hide value popover
)
```

## Field Layout

Use `class_wrapper` for Bootstrap grid layout:
```python
fields = dict(
    first=StringField(label="First name", class_wrapper="col-md-6"),
    last=StringField(label="Last name", class_wrapper="col-md-6"),
)
```

## Dynamic Fields

Fields can be computed dynamically based on player state:
```python
@classmethod
async def fields(page, player):
    return dict(
        offer=DecimalField(
            label="Your offer",
            min=0,
            max=player.endowment,
        ),
    )
```

## Rendering in Templates

```html
{{ field(form.amount) }}     {# Render a single field, including its errors #}
{{ fields() }}               {# Render all fields #}
{{ errors() }}               {# Form-level errors; needed for hand-written inputs and Likert macros #}
```

Field labels, descriptions, and choices are run through translation; see the
Internationalization section of `references/advanced.md`.
