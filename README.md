# skills

Agent skills that teach AI coding assistants how to add [Latitude](https://latitude.so) observability to applications using the official [Latitude Telemetry](https://github.com/latitude-dev/latitude-llm/tree/main/packages/telemetry) packages (OpenTelemetry-based).

## Skills

| Skill | Description |
| ----- | ----------- |
| [latitude-telemetry](skills/latitude-telemetry) | Add or audit `@latitude-data/telemetry` (TypeScript) and `latitude-telemetry` (Python) instrumentation: bootstrap (`initLatitude` / `init_latitude`), advanced OpenTelemetry integration (`LatitudeSpanProcessor`, `registerLatitudeInstrumentations`), `capture()` for user/session/tags, and Vercel / Next.js deployment patterns. |

## Installation

### Cursor (project)

Copy or symlink the skill into your repository:

```bash
git clone https://github.com/YOUR_ORG/skills.git
cd skills
ln -s "$(pwd)/skills/latitude-telemetry" /path/to/your/project/.cursor/skills/latitude-telemetry
```

Adjust the clone URL to match where you host this repo.

### Cursor (personal)

Symlink into your personal skills directory (path may vary by Cursor version):

```bash
ln -s /path/to/skills/skills/latitude-telemetry ~/.cursor/skills/latitude-telemetry
```

Do not install skills under `~/.cursor/skills-cursor/`; that tree is reserved for Cursor-managed built-ins.

### Manual copy

Copy `skills/latitude-telemetry/` into `.cursor/skills/latitude-telemetry/` in any project.

## Prerequisites

- A Latitude account and project.
- `LATITUDE_API_KEY` and `LATITUDE_PROJECT_SLUG` (and optionally `LATITUDE_TELEMETRY_URL` for ingest endpoint overrides).

Keys and project identifiers come from the Latitude product UI for your workspace.

## Upstream source of truth

Package READMEs and examples live in the monorepo:

- [packages/telemetry/typescript](https://github.com/latitude-dev/latitude-llm/tree/main/packages/telemetry/typescript)
- [packages/telemetry/python](https://github.com/latitude-dev/latitude-llm/tree/main/packages/telemetry/python)

When package behavior or APIs change, prefer fetching those READMEs over relying on stale skill text.
