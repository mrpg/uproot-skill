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
Decimal number input.
```python
DecimalField(label="Amount", min=0, max=cu("10"), places=2)
DecimalField(label="Price", min=0, max=C.MAX_PRICE, step="0.01")
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

### LikertField
Likert scale (radio buttons in a row).
```python
LikertField(
    label="I enjoy this task",
    min=1, max=5,
    label_min="Strongly disagree",
    label_max="Strongly agree",
)
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
IBAN input with validation.
```python
IBANField(label="Your IBAN")
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
{{ field(form.amount) }}     {# Render a single field #}
{{ fields() }}               {# Render all fields #}
{{ errors() }}               {# Show validation errors #}
```
