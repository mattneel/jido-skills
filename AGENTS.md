# Agent Instructions — jido-skills

This is the **jido-skills** repository — a collection of Agent Skills for the Jido Elixir ecosystem.

## Repository Structure

- `source/skills/` — Canonical skill sources (edit here)
- `.agents/`, `.claude/`, `.cursor/`, `.gemini/` — Generated provider outputs (DO NOT EDIT)
- `scripts/` — Build tooling

## Rules

- **NEVER** edit files in `.agents/`, `.claude/`, `.cursor/`, or `.gemini/` directly — they are generated from `source/` by the build system.
- To add or edit a skill: modify `source/skills/{name}/SKILL.md`, then run `bun run build`.
- Skill format follows the [agentskills.io](https://agentskills.io) specification.
- SKILL.md files require YAML frontmatter with `name` and `description` fields.
- Keep SKILL.md under 500 lines. Use a `reference/` subdirectory for deep documentation.
- Use `{{command_prefix}}`, `{{model}}`, `{{config_file}}` placeholders for multi-provider support.
- Run `bun run build` after any source changes to regenerate provider outputs.
- Run `bun test` to validate skills.

## Release Hygiene

- Do not modify `CHANGELOG.md`; release notes are generated from Git history during release, so keep changes focused on proper Conventional Commits.
