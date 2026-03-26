# stepwise-builder

A Claude Code skill that builds any project incrementally — one verified step at a time.

Instead of generating everything at once, it creates a detailed plan, gets your approval, then executes each step through a subagent and runs a smoke test before moving on. Errors are caught early, at the exact step that caused them.

## How it works

1. **Plan** — breaks your request into small, logical steps and saves them to `PLAN.md`
2. **Approve** — you review and adjust the plan before anything is written
3. **Execute** — each step runs in a dedicated subagent; smoke test must pass before the next step starts
4. **Track** — `PLAN.md` stays updated with the status of every step (`pending` → `done` / `blocked`)

## Install

```bash
mkdir -p ~/.claude/skills/stepwise-builder ~/.claude/commands && \
  curl -sL https://raw.githubusercontent.com/HEchternacht/stepwise-builder/main/skills/stepwise-builder/SKILL.md \
    -o ~/.claude/skills/stepwise-builder/SKILL.md && \
  curl -sL https://raw.githubusercontent.com/HEchternacht/stepwise-builder/main/commands/step.md \
    -o ~/.claude/commands/step.md
```

Or manually:

```bash
# Clone
git clone https://github.com/HEchternacht/stepwise-builder

# Copy skill
mkdir -p ~/.claude/skills/stepwise-builder
cp stepwise-builder/skills/stepwise-builder/SKILL.md ~/.claude/skills/stepwise-builder/

# Copy /step command
mkdir -p ~/.claude/commands
cp stepwise-builder/commands/step.md ~/.claude/commands/
```

## Usage

### Start a new project

Just describe what you want to build:

```
build a REST API in Python with user auth and a products endpoint
```

Claude will invoke `stepwise-builder`, generate `PLAN.md`, and wait for your approval before writing any code.

### Resume an existing plan

Use the `/step` command to pick up where you left off:

```
/step
```

This reads `PLAN.md` in the current project root and executes the next pending step.

## File structure

```
~/.claude/
├── skills/
│   └── stepwise-builder/
│       └── SKILL.md       ← skill definition
└── commands/
    └── step.md            ← /step slash command
```

## Requirements

- [Claude Code](https://claude.ai/code)

## License

MIT
