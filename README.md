# skills

Agent skills that teach AI coding assistants how to add [Latitude](https://latitude.so) observability to applications using the official [Latitude Telemetry](https://github.com/latitude-dev/latitude-llm/tree/main/packages/telemetry) packages (OpenTelemetry-based).

## Skills

| Skill | Description |
| ----- | ----------- |
| [latitude-telemetry](skills/latitude-telemetry) | Add or audit `@latitude-data/telemetry` (TypeScript) and `latitude-telemetry` (Python) instrumentation. Sends LLM traces (OpenAI, Anthropic, Bedrock, Vercel AI SDK, LangChain, LlamaIndex, …) to a Latitude project. Covers codebase discovery, bootstrap (`initLatitude` / `init_latitude`), advanced OpenTelemetry integration (`LatitudeSpanProcessor`, `registerLatitudeInstrumentations`), and `capture()` for user/session/tags context. |

## Installation

### Recommended

Run this in your project — no prior install needed:

```bash
npx skills add latitude-dev/skills --skill "latitude-telemetry"
```

Works with Claude Code, Cursor, and any other agent that reads skills from the standard locations.

### Manual symlink

Clone this repo and symlink the skill into your agent's skills directory:

```bash
git clone https://github.com/latitude-dev/skills.git /path/to/latitude-skills
ln -s /path/to/latitude-skills/skills/latitude-telemetry /path/to/skills-directory/latitude-telemetry
```

Common skills directories:

- Cursor (project): `<repo>/.cursor/skills/latitude-telemetry`
- Cursor (personal): `~/.cursor/skills/latitude-telemetry` (do **not** use `~/.cursor/skills-cursor/`; that tree is Cursor-managed)

### Manual copy

If symlinks are not an option, copy `skills/latitude-telemetry/` into the target skills directory.

## Prerequisites

- A Latitude account and project.
- `LATITUDE_API_KEY` and `LATITUDE_PROJECT_SLUG` (and optionally `LATITUDE_TELEMETRY_URL` for ingest endpoint overrides).

Keys and project identifiers come from the Latitude product UI for your workspace.

## Upstream source of truth

Package READMEs and examples live in the monorepo:

- [packages/telemetry/typescript](https://github.com/latitude-dev/latitude-llm/tree/main/packages/telemetry/typescript)
- [packages/telemetry/python](https://github.com/latitude-dev/latitude-llm/tree/main/packages/telemetry/python)

When package behavior or APIs change, prefer fetching those READMEs over relying on stale skill text.

