# Search Patterns for the Examples Repository

When looking for how to implement something specific, search across the
uproot-examples codebase. Here are common search terms organized by feature:

## Groups and Multiplayer
```bash
rg "GroupCreatingWait" -g "*.py"
rg "group_size" -g "*.py"
rg "after_grouping" -g "*.py"
rg "other_in_group" -g "*.py"
rg "others_in_group" -g "*.py"
rg "group.players" -g "*.py"
rg "find_one" -g "*.py"
```

## Synchronization
```bash
rg "SynchronizingWait" -g "*.py"
rg "all_here" -g "*.py"
rg "synchronize.*session" -g "*.py"
```

## Rounds
```bash
rg "Rounds\(" -g "*.py"
rg "Repeat\(" -g "*.py"
rg "player.round" -g "*.py"
rg "within\(round" -g "*.py"
rg "along\(\"round" -g "*.py"
rg "digest\(" -g "*.py"
```

## Forms and Fields
```bash
rg "fields = dict\(" -g "*.py"
rg "DecimalField|RadioField|IntegerField|StringField" -g "*.py"
rg "LikertField|BooleanField|SelectField" -g "*.py"
rg "BoundedChoiceField|IBANField|FileField" -g "*.py"
rg "stealth_fields" -g "*.py"
rg "handle_stealth_fields" -g "*.py"
```

## Validation
```bash
rg "def validate" -g "*.py"
rg "handle_stealth_fields" -g "*.py"
```

## Real-Time / WebSocket
```bash
rg "@live" -g "*.py"
rg "notify\(" -g "*.py"
rg "send_to\(|send_to_one\(" -g "*.py"
rg "reload\(" -g "*.py"
rg "spawn\(" -g "*.py"
rg "uproot.invoke" -g "*.html" -g "*.js"
rg "uproot.receive" -g "*.html" -g "*.js"
rg "onCustomEvent" -g "*.html" -g "*.js"
```

## Randomization
```bash
rg "Random\(" -g "*.py"
rg "shuffled" -g "*.py"
rg "Between\(" -g "*.py"
rg "Bracket\(" -g "*.py"
```

## Timeouts
```bash
rg "def timeout" -g "*.py"
rg "timeout_reached" -g "*.py"
rg "timeout =" -g "*.py"
```

## Treatments
```bash
rg "treatment" -g "*.py"
rg "TREATMENTS" -g "*.py"
```

## Constants and Currency
```bash
rg "class C:" -g "*.py"
rg "cu\(" -g "*.py"
rg "__export__" -g "*.py"
```

## Persistent Data Models
```bash
rg "um.Entry" -g "*.py"
rg "um.create_model" -g "*.py"
rg "um.add_entry" -g "*.py"
rg "um.filter_entries" -g "*.py"
rg "import uproot.models" -g "*.py"
```

## Chat
```bash
rg "chat.create" -g "*.py"
rg "chat.add_player" -g "*.py"
rg "chat\(" -g "*.html"
```

## Page Flow Control
```bash
rg "def show" -g "*.py"
rg "may_proceed" -g "*.py"
rg "allow_back" -g "*.py"
rg "move_to_page|move_to_end|transition_to_page|transition_to_end" -g "*.py"
rg "NoshowPage" -g "*.py"
rg "def page_order" -g "*.py"
```

## Dropout Handling
```bash
rg "watch_for_dropout" -g "*.py"
rg "handle_dropout|dropout" -g "*.py"
```

## Player Initialization
```bash
rg "def new_player" -g "*.py"
rg "def new_session" -g "*.py"
rg "def restart" -g "*.py"
rg "def language" -g "*.py"
rg "def api2" -g "*.py"
```

## Template Patterns
```bash
rg "field\(form\." -g "*.html"
rg "appstatic" -g "*.html"
rg "x-data|x-init|x-text" -g "*.html"
rg "uproot.currentPage" -g "*.js"
```

## JavaScript Simulation
```bash
rg "uproot.simulate.on" -g "*.js"
rg "sim.fill\|sim.choose\|sim.submit" -g "*.js"
rg "uproot.submit" -g "*.js"
```
