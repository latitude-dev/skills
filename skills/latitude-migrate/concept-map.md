# Latitude V1 to V2 concept map

| V1 concept | V2 equivalent | Notes |
| --- | --- | --- |
| Workspace | Organization | API keys are organization-scoped (Settings, Keys) |
| Project | Project | One project per agent or AI feature. Signals are scoped per project |
| Prompt (PromptL document) | A prompt template in your code | See `promptl-to-code.md` |
| Version control: drafts, published version, `versionUuid` | Git, plus `metadata.prompt_version` and a release tag on every trace | Experiments compare releases by tag |
| Playground | Your local dev loop, plus a Sandbox project for dev and staging traces | |
| AI Gateway, `prompts.run()`, `prompts.chat()` | Direct provider SDK call inside `capture()` | Removed; nothing on the Latitude side replaces them |
| `customIdentifier` on a run | `sessionId` and `userId` on `capture()` | |
| Logs | Traces and Spans | One trace per turn |
| `logs.create()` | Telemetry SDK, or trace imports from Langfuse, LangSmith, Braintrust | There is no V1 log import |
| LLM-as-judge evaluation (live) | Signal with an LLM-judge evaluation (`createSignal`, kind `judge`) | Collects forward from creation; default sampling 10 percent |
| Programmatic rule evaluation | Signal with a rule evaluation, or a custom script | `text_match`, `empty_output`, `output_length`, `json_output`, `metric` |
| Human-in-the-loop evaluation | Annotations | Failed annotations feed signal discovery |
| Composite scores | Scores | Custom pipelines submit scores through `createScore` |
| `evaluations.annotate()` | `annotations.create()` / `scores.create()` (SDK 9) | |
| Dataset (CSV, parameter columns, label column) | Dataset (input, output, expected output, metadata, custom columns) | Parameter columns become the `input` object, the label becomes `expectedOutput` |
| Save logs to dataset | Add traces to a dataset (`importDatasetRowsFromTraces`) | |
| Batch evaluation over a dataset | Local replay: simulations and regression tests, tagged `simulation` | Your runner, your CI |
| Experiments (prompt A vs B on a dataset) | Experiments (traffic slice A vs B, filtered on tags) | Exclude the `simulation` tag from production cohorts |
| Prompt suggestions | Agent dispatch | A signal wakes a coding agent with the evidence |
| Triggers (email, schedule) | Your scheduler or queue | Removed with the gateway |
| Webhooks, prompt integrations | Monitors, Slack notifications, agent-dispatch webhooks | |
| Latitude tools, agents, subagents | Your agent framework | Tool calls appear in the Tools view |
| Cache configuration | Provider prompt caching | The Cost view shows cache economics |
| `latitude-sdk` 5.x, `@latitude-data/sdk` 5.x | `latitude-telemetry` / `@latitude-data/telemetry` for tracing; `latitude-sdk` 9.x / `@latitude-data/sdk` 9.x for the API | SDK 6 and later are a rewrite |
| `gateway.latitude.so/api/v3` | `api.latitude.so/v1`, the MCP at `api.latitude.so/v1/mcp`, the `latitude` CLI | |
