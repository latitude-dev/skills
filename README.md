# Skills

Agent skills that teach AI coding assistants how to build with [Latitude](https://latitude.so).

## Skills

| Skill                                           | Description                                                                    |
| ----------------------------------------------- | ------------------------------------------------------------------------------ |
| [latitude-telemetry](skills/latitude-telemetry) | Add Latitude Telemetry instrumentation for TypeScript and Python applications. |

## Installation

### Quick start

Install a skill with the `skills` CLI:

```sh
npx skills add https://github.com/latitude-dev/skills --skill <skill-name>
```

### Manual installation

Copy the desired skill's `SKILL.md` from `skills/<skill-name>/SKILL.md` into your agent harness's skills directory.

- **Claude Code**: add it under `.claude/skills/<skill-name>/SKILL.md` in your project, or under your user-level Claude Code skills directory if you want it available across projects.
- **Codex**: add it under `.codex/skills/<skill-name>/SKILL.md` in your project, or the equivalent user-level Codex skills directory.
- **Cursor**: add it under `.cursor/skills/<skill-name>/SKILL.md` in your project so Cursor agents can load it with the rest of your workspace instructions.
- **OpenCode**: add it under `.opencode/skills/<skill-name>/SKILL.md` in your project, or the equivalent user-level OpenCode skills directory.
- **Other `.agents`-compatible agent harnesses**: add it under `.agents/skills/<skill-name>/SKILL.md`.

After copying the skill, restart or reload your agent harness so it can discover the new instructions.
