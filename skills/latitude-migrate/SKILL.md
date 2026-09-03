---
name: latitude-migrate
description: Move an application from Latitude V1 (prompt manager, PromptL, the gateway's `prompts.run` / `prompts.chat`, `latitude-sdk` or `@latitude-data/sdk` below 6) to Latitude V2 (prompts in code, direct provider calls, telemetry, signals, datasets). Use when the user mentions Latitude V1, PromptL, the Latitude gateway, `LATITUDE_PROJECT_ID` / `LATITUDE_VERSION_UUID`, or asks to migrate, upgrade, or move to Latitude V2. Orchestrates `latitude-telemetry` (instrumentation) and `latitude-cli` (API access); read both.
---

# Latitude migrate (V1 to V2)

Latitude V1 hosted prompts (PromptL) and ran them through a gateway; the app called `prompts.run()`. Latitude V2 observes an agent the app runs itself: prompts live in the codebase, the app calls the model provider directly, and Latitude sees traffic through OpenTelemetry. This skill takes an app from one to the other and leaves it verified: real traces in a V2 project, the V1 evaluations recreated as signals, the V1 datasets imported, and a regression replay that runs locally.

The human-readable version of this flow is the docs page [Migrate from Latitude V1](https://docs.latitude.so/getting-started/migrate-from-v1). Read `concept-map.md` in this folder for the V1 to V2 mapping and `promptl-to-code.md` for the prompt conversion rules.

## Preflight

Confirm the app is on V1 before doing anything. Any of these is enough:

- `latitude-sdk` below 6 in `requirements.txt` / `pyproject.toml`, or `@latitude-data/sdk` below 6 in `package.json`.
- Calls to `prompts.run`, `prompts.chat`, `prompts.get`, `prompts.getAll`, `runs.*`, `logs.create`, `evaluations.annotate`, or HTTP requests to `gateway.latitude.so/api/v3`.
- `LATITUDE_PROJECT_ID` or `LATITUDE_VERSION_UUID` in env or config. V2 uses `LATITUDE_PROJECT_SLUG`.
- `.promptl` files, or `{{ variables }}` with a `provider:` / `model:` frontmatter block.

Then find out what V2 access exists, in this order: a connected and authenticated Latitude MCP; a `LATITUDE_API_KEY` that works against `https://api.latitude.so/v1/account`; nothing. With nothing, hand the account creation to `latitude-setup` (temporary account and claim link) and come back here. Do not run `latitude-setup`'s delete-and-recreate cleanup on a project the user already owns.

You also need the V1 API key and V1 project id (from the app's env). If the V1 key returns 402 or 401, the V1 workspace has lapsed; ask the user for a UI export of prompts and datasets, or reconstruct the prompts from the call sites and tell the user that is what you did.

## Ground rules

- **Plan, then wait.** Present the plan (template below) and get an unambiguous `go ahead` before editing app code. One material question at a time before that.
- **Never print secrets.** V1 and V2 keys go into the app's existing secret store or `.env`; never into chat, logs, or commits.
- **Do not delete anything in V1.** The V1 workspace stays as a fallback until the user retires it (last step, user's call).
- **Keep the app runnable at every step.** Do the work on a branch. The provider call replaces the gateway call one use case at a time; parity before cleanup.
- **V2 is a new project, not a data migration.** Traces, signals, and evaluations rebuild from live traffic. Only prompts and datasets carry over as files.
- **Do not memorize the API.** MCP tools and CLI commands are generated from the same API; discover shapes with the MCP tool schemas or `latitude <resource> <command> --schema`.

## Workflow

### 1. Inventory V1 into `latitude-v1-export/`

Pull everything the V1 API still serves, with the V1 key as `Authorization: Bearer`:

```bash
V1=https://gateway.latitude.so/api/v3
# every prompt at the version the app pins (use `live` when LATITUDE_VERSION_UUID is unset)
curl -s -H "Authorization: Bearer $LATITUDE_V1_API_KEY" "$V1/projects/$LATITUDE_PROJECT_ID/versions/${LATITUDE_VERSION_UUID:-live}/documents"
# datasets and their rows (paginated)
curl -s -H "Authorization: Bearer $LATITUDE_V1_API_KEY" "$V1/datasets?page=1&pageSize=100"
curl -s -H "Authorization: Bearer $LATITUDE_V1_API_KEY" "$V1/dataset-rows?datasetId=<id>&page=1&pageSize=100"
```

Write each document's `content` to `latitude-v1-export/prompts/<path>.promptl`, each dataset to `latitude-v1-export/datasets/<name>.csv` (one column per row key; keep the label column name), and commit the folder. Evaluations are not on the V1 API: ask the user for each evaluation's name, type (LLM judge, programmatic rule, human), criteria, and whether it ran live, and write them to `latitude-v1-export/evaluations.md`. Ask once, with the prompt list in hand so the user can answer per prompt.

### 2. Map the call sites

For every gateway call record: the prompt path, the parameters passed, what the app does with the response (`response.text`, tool calls, streaming events, `uuid`), the `customIdentifier` if any, and the PromptL features the document uses (frontmatter keys, conditionals, loops, `<step>` chains, tools, `type: agent`, `schema`, snippets). Group call sites by use case, the way `latitude-telemetry` groups them. One use case becomes one `capture()` boundary and one prompt module.

### 3. Present the plan and wait

```text
Plan
- V1 inventory: N prompts (paths), M datasets, evaluations found: ...
- Use cases: one line each (entry point, prompt, provider/model from the frontmatter, what the response feeds)
- Prompt conversion: where each prompt module goes, which PromptL features need hand-written code (chains, tools, agent loops), and the version string each prompt carries
- Provider calls: which SDK replaces the gateway call per use case, and the response-shape changes the app must absorb
- Telemetry: bootstrap location, capture() boundaries, user/session source, tags (release) and metadata (prompt_version, release)
- V2 project: existing slug or new project name; where LATITUDE_API_KEY and LATITUDE_PROJECT_SLUG will live
- Control plane: one signal per V1 evaluation (judge or rule), one dataset per V1 dataset with the label mapped to expected output
- Verification: the real flow that will run, and how traces will be read back (MCP, CLI, or API)
- Not touched: the V1 workspace, until you retire it

Reply `go ahead` to approve this plan.
```

### 4. Move the prompts into code

Follow `promptl-to-code.md`. Each prompt becomes a module in the app's language with a version string (`grammar-check@2`) that will travel as `metadata.prompt_version`. Frontmatter keys become provider-call arguments; check every key against the provider SDK version the app installs, because providers drop and rename parameters (the Anthropic Python SDK 1.x has no `temperature` on `messages.create()`). Keep the `.promptl` originals in the export folder until parity is confirmed.

### 5. Replace the gateway call with a provider call

Per use case: install the provider SDK if the app does not have it, build the messages from the prompt module, call the provider, and adapt the response handling (`result.response.text` becomes the provider's content accessor; V1 streaming `onEvent` callbacks become the provider's stream). Carry `customIdentifier` into a `session_id` or `user_id` for the next step. Remove the `Latitude(...)` SDK 5 client construction once no use case needs it.

### 6. Instrument (delegate to `latitude-telemetry`)

Use `latitude-telemetry`'s "invoked directly" entry point. Requirements for this migration: construct the telemetry `Latitude(...)` once, before the provider client, with the app's own provider module; one `capture()` per use case with `user_id`, `session_id`, a `release-<version>` tag plus a `production` tag, and metadata `{prompt_version, release}`; flush before short-lived processes exit. Tags are the cohort key for session filters and Experiments; metadata is for exact values and trace filters.

### 7. Create the V2 project and configure the app

If no target project exists, create one per agent or AI feature: MCP `createProject` or `latitude projects create --name "<App>"`. Write `LATITUDE_PROJECT_SLUG` and `LATITUDE_API_KEY` where the app keeps config (update existing `LATITUDE_*` entries in place), and remove `LATITUDE_PROJECT_ID` and `LATITUDE_VERSION_UUID`.

### 8. Recreate the control plane

Do this **before** generating verification traffic; signal detectors collect forward from creation and never backfill.

- **Each V1 evaluation becomes a signal.** LLM-as-judge: `createSignal` with `evaluation.settings.kind = "judge"` and the V1 instructions rewritten as pass/fail criteria in plain prose (no backticks; the criteria is compiled and backticks break it). Programmatic rule: `kind = "rule"` with `text_match`, `json_output`, `empty_output`, `output_length`, or `metric` conditions. Human-in-the-loop: no object to create; tell the user annotations on traces replace it. Scope each signal with a `filters` pre-gate on the use case's tag. User-defined judge signals default to 10 percent sampling; tell the user to raise it in the signal's scope for low-traffic projects.
- **Each V1 dataset becomes a dataset.** `createDataset`, then `insertDatasetRows` with `input` = an object of the parameter columns, `expectedOutput` = the label column, `metadata.source` = the V1 dataset name. Add a custom column with `addDatasetColumn` when the V1 dataset had extra columns worth keeping.
- **Batch experiments become a local replay** (step 10). Live V1 experiments comparing prompt versions become an Experiment over release tags once two releases have traffic.

### 9. Verify with real traffic

Run the user's real flow once per use case. Read the traces back through the MCP (`listTraces`, `getTrace`), the CLI (`latitude traces list --project-slug <slug> --format json`), or the API. Confirm per trace: the model and token counts are non-zero, the conversation renders as system, user, assistant, `userId` and `sessionId` are set, and `metadata.prompt_version` and `metadata.release` carry what you set. Loop until correct. Do not delete the project to clean up; these traces are the user's first V2 data.

### 10. Regression replay

Write a test that pulls the dataset with the V2 SDK (`client.datasets.list_rows` / `client.datasets.listRows`), replays each row's input through the use case, and compares against `expectedOutput`. Tag every replay `simulation` and `agent-<release>` and put the dataset slug and row id in metadata. Exact-match labels are brittle: when a replay produces an equally valid answer, extend the row (custom column such as `alsoAccepts`) rather than loosening the test. Exclude the `simulation` tag from experiment cohorts and monitors that should only see real users.

### 11. Hand back

Report: the V2 project slug and console URL, the traces verified, the signals created (slugs) and their sampling, the datasets and row counts, how to run the replay, and the decommission checklist: remove the SDK 5 dependency and every `prompts.*` / `logs.create` / `evaluations.*` call, delete `LATITUDE_PROJECT_ID` and `LATITUDE_VERSION_UUID`, rotate the V1 key, keep `latitude-v1-export/` until the user deletes it.

## Gotchas seen in real migrations

- Sessions belong to one release. Reusing session ids across deploys creates sessions no cohort filter can own; start new sessions on deploy or suffix the id.
- Session filters match on tags; put the release in a tag as well as in metadata.
- Judge criteria with backticks are rejected with a syntax error.
- Short scripts and tests lose their last batch of spans without `flush()` and `shutdown()`.
- The MCP is OAuth-only; an API key on `api.latitude.so/v1/mcp` gets a 401. Use the key for the SDK, CLI, and REST.
- Coding agents sometimes pull cached V1 docs. Use `https://docs.latitude.so/llms.txt` as the documentation index.
