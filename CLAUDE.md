# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Claude Code plugin that provides skills for managing cloud infrastructure. The plugin is registered via `.claude-plugin/plugin.json`, which points to `skills/` and `commands/` directories.

Skills are Markdown files (`SKILL.md`) that Claude Code reads to gain domain knowledge. They are **not executable code** — they contain instructions, patterns, and reference pointers that shape how Claude responds to infrastructure-related prompts.

## Repository Structure

```
.claude-plugin/
  plugin.json        # Plugin manifest — registers skills/ and commands/
  marketplace.json   # Marketplace listing metadata

skills/
  <skill-name>/
    SKILL.md         # Trigger conditions + core instructions for Claude
    references/      # Deep-dive documentation that SKILL.md links to
```

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with a YAML front matter block:
   ```markdown
   ---
   name: skill-name
   description: <natural language triggers that cause this skill to activate>
   ---
   ```
2. The `description` field is critical — it determines when the skill auto-triggers. Write it as a list of user intents and keywords, not a feature summary.
3. Add `references/` files for deep content that would bloat `SKILL.md`. Reference them by relative path from within `SKILL.md`.
4. No registration step needed — `plugin.json` points to `skills/` as a directory.

## Skill Authoring Conventions

- `SKILL.md` should contain: trigger conditions, core philosophy, workflow steps, quick-start patterns, and pointers to `references/` files.
- `references/*.md` files are for complete examples, decision matrices, and content too long for `SKILL.md`.
- When referencing a `references/` file from `SKILL.md`, use a relative path like `references/state-management.md`.
- The `terraform-specialist` skill is the canonical example — use it as a template for structure and depth.

## Plugin Manifest

`.claude-plugin/plugin.json` fields that matter:
- `"skills": "../skills/"` — path to skills directory (relative to the manifest)
- `"commands": "../commands/"` — path to slash commands directory (not yet populated)
- `"version"` — bump this when publishing updates