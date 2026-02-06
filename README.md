# Build — Claude Code Plugin

A Claude Code plugin for software development workflows. Currently includes an interactive PRD builder, with more skills planned.

## Install

In any Claude Code session:

```
/plugin install github:ryanismert/claude-plugin-build
```

## Available Skills

### `/build:prd` — Product Requirements Document Builder

Takes a short product idea, asks focused clarifying questions across 3–5 rounds, then generates a complete PRD markdown file in your project.

```
/build:prd A browser extension that helps colorblind users distinguish UI elements
```

Or just `/build:prd` and describe your idea when prompted.

**Features:**
- Iterative Q&A — questions come in small batches, not all at once
- Adaptive — skips questions you've already answered, respects "skip" and "that's everything"
- Structured output — saves to `docs/prd-<product-name>.md` using a consistent template
- Assumptions marked — any gaps are flagged as **[TBD]** or **[ASSUMPTION]**

## Plugin Structure

```
claude-plugin-build/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
└── skills/
    └── prd/
        ├── SKILL.md             # Skill definition + prompt
        └── references/
            └── prd-template.md  # Output template
```

## Customization

- **Edit the template** (`skills/prd/references/prd-template.md`) to match your team's PRD format
- **Edit SKILL.md** to change question rounds or conversation style
- **Add new skills** by creating new directories under `skills/`

## Uninstall

```
/plugin
```

Select the plugin and choose "Uninstall".
