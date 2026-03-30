# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a skill repository for Claude Code. Skills are reusable guides that Claude Code instances invoke via the `Skill` tool to implement specific architectural patterns. The repository contains no build system or tests — it is purely documentation.

## Skill Structure

Each skill lives at `[domain]/[skill-name]/SKILL.md`. The SKILL.md file must begin with YAML frontmatter:

```yaml
---
name: skill-identifier          # matches the directory name
description: Use when...        # triggers for when Claude should invoke this skill
---
```

After the frontmatter, SKILL.md contains the full implementation guide: directory structure, code snippets, step-by-step instructions, and a "Common Mistakes" table. Skills should be self-contained — a Claude instance reading only the SKILL.md should have everything needed to implement the pattern.

## Domain Organization

Skills are organized by domain:

```
frontend/     ← UI frameworks, build tooling, CSS patterns
backend/      ← (future) server-side patterns
```

## Writing Skills

When creating or editing a skill:

- The `description` frontmatter field is what Claude uses to decide whether to invoke the skill — make it concrete and trigger-focused
- Include a "Common Mistakes" table at the end documenting known gotchas and their fixes
- Code examples should be copy-paste ready and complete (no ellipsis or "fill this in")
- Document Windows-specific caveats when shell behavior differs (e.g., env var syntax)
- Keep platform-specific workarounds inline in the relevant section rather than in a separate appendix
