# uproot skill for agentic coding models

A skill for building browser-based behavioral experiments with the [uproot](https://uproot.science/) framework.

## What this skill does

When activated, it guides agentic coding models such as Claude Code, Codex, or similar through the full workflow of building uproot experiments:

- Researching existing examples before writing code
- Structuring apps correctly (Python logic, HTML templates, simulate.js)
- Using the right page types, field types, and lifecycle hooks
- Building game theory experiments, surveys, real effort tasks, and more
- Writing proper templates with Bootstrap 5 and Alpine.js
- Setting up automated testing with simulate.js

## Installation (if used with Claude Code)

### As a project skill

Copy the skill directory, `uproot`, into your project's `.claude/skills/`:

```bash
cp -r uproot /path/to/your/project/.claude/skills/
```

### As a user-level skill

Copy to your Claude home directory:

```bash
mkdir -p ~/.claude/skills
cp -r uproot ~/.claude/skills/
```

## Reference materials

The `references/` directory contains detailed documentation on:

- **page-types.md** - Page classes, lifecycle methods, hooks
- **field-types.md** - All available field types and their parameters
- **template-patterns.md** - HTML/Jinja2 templates, JavaScript, Alpine.js
- **game-theory.md** - Patterns for game theory experiments
- **page-ordering.md** - Rounds, Between, Random, Bracket constructs
- **search-patterns.md** - Grep patterns for finding features in examples
- **advanced.md** - Models, live methods, chat, dropouts, and more

## Examples repository

This skill works best alongside the [uproot-examples](https://github.com/mrpg/uproot-examples) repository, which contains many complete example apps covering most common experimental paradigms. The skill will cause LLM agents to clone `uproot-examples` if `git` is available.

## License

CC0
