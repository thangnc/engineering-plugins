# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A monorepo of Claude Code plugins intended for deployment on the Claude Code marketplace. Each plugin lives under `plugins/` and bundles one or more skills.

Skills are Markdown files (`SKILL.md`) that Claude Code reads to gain domain knowledge. They are **not executable code** — they contain instructions, patterns, and reference pointers that shape how Claude responds to domain-specific prompts.

## Repository Structure

```
plugins/
  <plugin-name>/
    .claude-plugin/
      plugin.json        # Plugin manifest — registered with the marketplace
    skills/
      <skill-name>/
        SKILL.md         # Trigger conditions + core instructions for Claude
        references/      # Deep-dive documentation that SKILL.md links to

.claude-plugin/
  marketplace.json       # Top-level marketplace listing metadata
```

### Current plugins

| Plugin | Path | Skills |
|---|---|---|
| `cloud-devops` | `plugins/cloud-devops/` | `terraform-specialist` |

## Marketplace Deployment

This repo is structured for direct marketplace publication.

### Manifest: `plugins/<plugin-name>/.claude-plugin/plugin.json`

This is the file Claude Code reads when installing the plugin. Key fields:

```json
{
  "name": "<plugin-name>",
  "version": "1.0.0",
  "skills": "./skills/"
}
```

- `"skills"` — path to the skills directory, relative to `plugin.json` (i.e. `plugins/<plugin-name>/skills/`)
- `"version"` — bump this on every marketplace publish
- `"repository"` — must point to the public GitHub URL

### Marketplace metadata: `.claude-plugin/marketplace.json`

Controls how the plugin appears in the marketplace listing. Update `metadata.description` and `plugins[].keywords` to improve discoverability.

### Publishing a new version

1. Make your changes under `plugins/<plugin-name>/`
2. Bump `"version"` in `plugins/<plugin-name>/.claude-plugin/plugin.json`
3. Update `"source"` in `.claude-plugin/marketplace.json` if the plugin path changed
4. Commit and push to `main`
5. The marketplace will pick up the new version from the GitHub URL registered in the manifest

## Adding a New Plugin

1. Create `plugins/<plugin-name>/.claude-plugin/plugin.json` with `"skills": "./skills/"`
2. Create `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`
3. Add the new plugin to `.claude-plugin/marketplace.json` under the `"plugins"` array with its `"source"` pointing to `./plugins/<plugin-name>`

## Adding a New Skill

1. Create `plugins/<plugin-name>/skills/<skill-name>/SKILL.md` with a YAML front matter block:
   ```markdown
   ---
   name: skill-name
   description: <natural language triggers that cause this skill to activate>
   ---
   ```
2. The `description` field is critical — it determines when the skill auto-triggers. Write it as a list of user intents and keywords, not a feature summary.
3. Add `references/` files for deep content that would bloat `SKILL.md`. Reference them by relative path from within `SKILL.md`.
4. No registration step needed — `plugin.json`'s `"skills"` path covers all subdirectories.

## Skill Authoring Conventions

- `SKILL.md` should contain: trigger conditions, core philosophy, workflow steps, quick-start patterns, and pointers to `references/` files.
- `references/*.md` files are for complete examples, decision matrices, and content too long for `SKILL.md`.
- When referencing a `references/` file from `SKILL.md`, use a relative path like `references/state-management.md`.
- The `terraform-specialist` skill is the canonical example — use it as a template for structure and depth.