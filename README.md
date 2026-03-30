# claude-skills

A collection of reusable skills for [Claude Code](https://claude.ai/code). Each skill is a self-contained implementation guide that Claude Code instances invoke via the `Skill` tool to implement specific architectural patterns.

## Structure

Skills are organized by domain:

```
[domain]/
└── [skill-name]/
    └── SKILL.md    ← metadata + full implementation guide
```

## Available Skills

### Frontend

| Skill | Description |
|-------|-------------|
| [vite-multi-tenant-setup](frontend/vite-multi-tenant-setup/SKILL.md) | Convert a Vite + React app to multi-tenant architecture with isolated Tailwind themes, Vite configs, and npm scripts per tenant |

## Adding a Skill

1. Create a directory under the appropriate domain: `[domain]/[skill-name]/`
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`) followed by the implementation guide
3. List the skill in this README

See [CLAUDE.md](CLAUDE.md) for authoring guidelines.
