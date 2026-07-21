# Skills

Agent skills that teach AI coding assistants how to build with [Latitude](https://latitude.so).

## Skills

| Skill                                                         | Description                                                                                                                                     |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| [latitude-setup](skills/latitude-setup)                       | Zero-account onboarding: bootstrap a temporary Latitude account, instrument, verify traces, claim.                                              |
| [latitude-cli](skills/latitude-cli)                           | Install, authenticate, and drive the `latitude` CLI to run Latitude API operations from a terminal.                                             |
| [latitude-telemetry](skills/latitude-telemetry)               | Add Latitude Telemetry instrumentation for TypeScript and Python applications.                                                                  |
| [latitude-memory-telemetry](skills/latitude-memory-telemetry) | Add Latitude observability for an agent's long-term memory (files, DB, vector store, or a provider) — per-record history, diffs, and provenance. |

> **`latitude-setup` builds on the other two.** It orchestrates `latitude-cli` (install + auth) and `latitude-telemetry` (instrumentation). The `skills` CLI does not auto-install dependency skills, so add all three when you want the from-scratch onboarding flow:
>
> ```sh
> npx skills add https://github.com/latitude-dev/skills --skill latitude-setup,latitude-cli,latitude-telemetry
> ```
>
> **`latitude-memory-telemetry` is a telemetry add-on.** It builds on `latitude-telemetry` and applies only when the app has long-term memory (state persisted across sessions, runs, or users). Add it alongside `latitude-telemetry` for those apps:
>
> ```sh
> npx skills add https://github.com/latitude-dev/skills --skill latitude-telemetry,latitude-memory-telemetry
> ```

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
