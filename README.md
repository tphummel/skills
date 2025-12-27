# Skills

A collection of skills following the [agentskills.io](https://agentskills.io/) specification.

## Structure

Each subdirectory represents a skill and contains a `SKILL.md` file with frontmatter matching the directory name.

## Usage

Skills are packaged as `.zip` files via GitHub Actions and available as artifacts. Each skill can be consumed by Claude Code, Codex, or other AI tools.

## Skill Format

Each skill directory must contain a `SKILL.md` file with the following structure:

```markdown
---
name: skill-name
description: A description of what this skill does and when to use it.
---

[Skill content here]
```

The directory name must match the `name` in the frontmatter.
