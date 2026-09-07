---
name: latitude-setup
description: Zero-account onboarding orchestrator for Latitude. Bootstrap a temporary Latitude account from the terminal (no signup), instrument the app or agent harness (Claude Code, Hermes, OpenClaw, Pi, Prime Intellect) for tracing, verify real traces with the Latitude CLI, clean up, and hand back a browser link to claim ownership together with a first Artifact, an HTML page showing what the telemetry captured. Use when someone wants to set up Latitude, install Latitude telemetry, or "try Latitude" and may have no account or API key yet — the landing-page "try it with your agent" flow. If it turns out the user already has an account, API key, or connected Latitude MCP, this skill redirects to latitude-telemetry + latitude-cli instead of creating a temporary account.
---

# Latitude Setup (zero-account onboarding)

## Outcome

You are done when the user has, in one final message from you:

1. Telemetry installed in their app or agent harness and **verified with the Latitude CLI** against a real run.
2. **A first Artifact**: `artifacts/first-session.html`, built from the bundled template with the data of the clean session you just verified, so the user sees what Latitude captured (model calls, tool calls, tokens, cost, timing, the conversation) without opening anything else. When the account is temporary, the page carries the **Claim your workspace** button.
3. The claim link and its expiry (temporary-account flow), and any restart the user still owes.

Installing telemetry is the prerequisite. The Artifact is how the user sees the value in the first five minutes; it is part of the deliverable, built by default, and skipped only when the user explicitly says so. Verifying traces and building the Artifact are two separate steps (8 and 9): verification proves the pipeline with the CLI, the Artifact shows the user what came through.

This skill orchestrates the **from-scratch** path: the user has **no Latitude account and no API key**. It provisions a temporary account via the CLI, instruments the target, verifies real traces, cleans up, builds the Artifact, and returns a claim link.

The **target** is whatever should emit traces: an **app** (instrumented with the Latitude SDKs or an OTLP exporter) or an **agent harness** the user runs locally (Claude Code, Hermes, OpenClaw, Pi, Prime Intellect, each with its own plugin). Everything below applies to both; steps call out where a harness differs.

This skill depends on **`latitude-cli`** (install + auth + command primitives) and **`latitude-telemetry`** (instrumentation). Read both; this skill adds the orchestration between them. The first Artifact needs nothing beyond this skill's own `first-artifact.html` template and the CLI. For richer or refreshable reports later, the separate `latitude-artifacts` skill exists; it is not needed here.

## Preflight: existing account, or create one? Decide in one question, never by asking for a key

The temporary-account bootstrap exists so that someone with **nothing set up yet** gets going without a signup. A user who already has a Latitude account can also land here, and routing them through a redundant temporary org is worth avoiding. Decide which case you are in with the rules below. **Whatever you find, this skill never ends a turn with "I need your `LATITUDE_API_KEY` and project slug".** Asking for a key is the one outcome that defeats the purpose: the key either exists already, or the bootstrap creates it, or the user tells you they have an account and hands it over on their own initiative.

**The only hard signals that an account exists:**

- **A working API key is already present**: `LATITUDE_API_KEY` with a value in the shell, in a `.env` (search the app root and its parents), in the app's secret manager, in deployment/CI config, or in a harness's own config (`~/.hermes/.env`, the `LATITUDE_*` env block in `~/.claude/settings.json`, `~/.pi/agent/latitude-telemetry.json`, the `Authorization` header in the `diagnostics.otel` block of `~/.openclaw/openclaw.json`).
- **A Latitude MCP is connected and authenticated** in this harness: an OAuth-authorized Latitude MCP means the user already has a workspace.
- **The user says so**: they mention being signed in, having a project, or already using Latitude.

**What is not a signal:** a project slug, a `LATITUDE_PROJECT` setting, an enabled plugin, a `@latitude-data/telemetry` / `latitude-telemetry` dependency, or an OTLP exporter pointed at `ingest.latitude.so` **without a key next to it**. That is a placeholder, an example, or a leftover from an earlier attempt. It tells you where to write the values, not that an account exists. Do not turn it into "the setup is half done, give me the key".

**If a hard signal holds → do NOT bootstrap.** The user has an account; take the direct path:

1. Use **`latitude-telemetry`** via its "invoked directly" entry point to audit and instrument the app against the existing key/project. Use **`latitude-cli`** if you still need to install/authenticate the CLI or discover the project slug (`latitude projects list`).
2. Finish with `latitude-telemetry`'s verification step: run the user's **real** LLM flow and confirm the traces landed via the Latitude **MCP, CLI, or API**. Do **not** run this skill's bootstrap, claim-link, or delete-and-recreate cleanup: those belong only to the temporary-account flow, and the cleanup would destroy a project the user actually owns.
3. Build the first Artifact exactly as in step 9 (without the `claim` block) and hand everything back as in step 10. The Outcome above applies to this path too.

**If no hard signal holds and the user is present → ask one question, then act.** For a harness target, ask it in the same message as the content-consent question and the Artifact line from step 4 (one message, all decisions, before anything is created or installed). The temporary account is the default:

```text
Do you already have a Latitude account?
  (a) No, create a temporary one for me now, no signup, I'll claim it later   (default)
  (b) Yes, I'll give you its API key and project slug
```

On (a), or no answer, or "just do it": bootstrap (step 2). On (b): wait for the values, put them where step 3 says, and take the direct path above. Do not ask any other question about accounts, keys or projects.

**If no hard signal holds and the user is not there to answer → bootstrap.** A bootstrapped temporary organization is low-stakes and recoverable: the claim link lets it be redeemed later, and it can be claimed even into an account the user already has, so creating one for a user who turned out to already have an account is a minor, fixable outcome, not a failure. The overriding goal is to get the target instrumented and emitting real traces to Latitude **autonomously**; never block that on a missing key.

## Ground rules

- **Completion contract.** Setup is **not done when traces arrive**, and not done when the claim link is delivered. It is done when the final message (step 10) contains: what was configured, the verified real trace, the claim link and its expiry (temporary-account flow), any restart the user still owes, **and the first Artifact** (its path and what it shows), or, only if the user struck the Artifact, a one-line offer to build it later. An ending without one of those two is an incomplete run, the same as ending before verification. Do not treat "traces verified" or "claim link sent" as the finish line.
- **The Artifact is decided up front, not at the end.** It appears in the plan the user approves (app target) or in the consent question (harness target) as default scope, so there is no second question at the end and nothing to forget.
- **Plan, then wait.** Instrumentation goes through `latitude-telemetry`'s "present a plan, wait for explicit approval" contract. Do not edit app code before approval.
- **Never print raw secrets.** The bootstrap API key must never appear in chat, logs, commits, or the Artifact. The claim link is safe to show (it's the whole point); the API key is not.
- **One project, named once.** Bootstrap creates exactly one project. There is no throwaway "testing" project; cleanup is delete + recreate with the same name (step 7).
- **Verify with the CLI, not by assumption.** `latitude traces list`, `latitude traces listSpans` and `latitude traces getSpan` are the evidence. Their output is also the data you paste into the Artifact.

## The flow

Ten steps. Each one ends with something concrete in hand; do not move on without it.

### 1. Install dependencies

**Do:** make sure the `latitude-telemetry` and `latitude-cli` skills are available, and install the `latitude` binary as described in **`latitude-cli` → Install** (OS/ARCH detection, download the matching release asset, place on `PATH`, `chmod +x`). **Only `cli-5.0.0` and later are the real Latitude CLI**; `latitude-cli` explains why earlier `cli-*` tags must be ignored.

**In hand:** `latitude --version` prints 5.0.0 or later.

### 2. Bootstrap a temporary account

This is unauthenticated; no key needed yet. Infer a sensible project name from the app or harness (for a harness, its name works: "hermes", "claude-code") and optionally an org name. Ask the user for an email only if you want the claim link mailed to them.

**First, discover the command's exact flags and response fields.** Don't assume the shape; it can change across versions:

```bash
latitude account bootstrap --schema
```

**Keep the API key out of your own conversation.** `--format json` prints the API key to stdout, which lands in the transcript. Capture the response to a scratch file instead of letting it print, then display everything *except* the secret:

```bash
latitude account bootstrap \
  --project-name "<inferred project name>" \
  --organization-name "<inferred org name>" \
  --format json > ~/.latitude-bootstrap.json
# optional: --user-email <address>  (also emails the claim link)

# Show the non-secret fields; drop the API key (and any other sensitive field --schema flags):
jq 'del(.apiKey)' ~/.latitude-bootstrap.json
```

The flow relies on a few of the response fields, by role (confirm their exact names in `--schema`):

- the **API key**: an org-scoped secret; never print it. It flows straight into `.env` in step 3, then the scratch file is deleted.
- the **project slug**: the single created project; your telemetry target and the cleanup handle (step 7).
- the **claim link**, plus its expiry and the email it was sent to if you passed one: it goes into the Artifact (step 9) and the final message (step 10). Keep these three non-secret values somewhere you can read them later (a note, or the `jq 'del(.apiKey)'` output saved to a non-secret file).

**In hand:** project slug, claim link, expiry; the API key on disk in the scratch file only.

### 3. Configure auth via `.env`

**Do:** write the key and project slug into the app's `.env` **straight from the scratch file, so the value never prints**. Make sure `.env` is gitignored. Replace any existing `LATITUDE_*` lines rather than duplicating them, then delete the scratch file:

```bash
touch .env
# Drop any prior LATITUDE_API_KEY / LATITUDE_PROJECT_SLUG lines (avoids duplicates). BSD+GNU sed:
sed -i.bak '/^LATITUDE_API_KEY=/d;/^LATITUDE_PROJECT_SLUG=/d' .env && rm -f .env.bak
# Append the real values from the file, never echoed (field names .apiKey/.projectSlug per --schema):
printf 'LATITUDE_API_KEY=%s\n'     "$(jq -r .apiKey      ~/.latitude-bootstrap.json)" >> .env
printf 'LATITUDE_PROJECT_SLUG=%s\n' "$(jq -r .projectSlug ~/.latitude-bootstrap.json)" >> .env
# Remove the scratch file so the key isn't left on disk:
rm -f ~/.latitude-bootstrap.json
```

If any value you later add to `.env` contains spaces (not the key/slug; e.g. a header/token value), wrap it in double quotes: the CLI's `.env` parser rejects unquoted spaced values and stops, after which it won't read `LATITUDE_API_KEY` (see `latitude-cli` → Authentication).

`LATITUDE_API_KEY` authenticates **both** the `latitude` CLI (it auto-loads `.env` from the working directory and its parents) **and** the telemetry SDK: one entry, both consumers. **Do not** run `latitude auth login`; it invokes the OS keychain and can block on a prompt. Run subsequent `latitude` commands from the app root so `.env` is picked up. Do not echo the key back to the user.

**Harness target:** the harness does not read the app's `.env`. Copy the two values (never echoed, same `jq` pattern) to wherever that harness reads them, per its docs page: Hermes takes `LATITUDE_API_KEY` and `LATITUDE_PROJECT` in `~/.hermes/.env`; Claude Code and Pi take them as installer flags (`--api-key`, `--project`); OpenClaw takes them as headers in `~/.openclaw/openclaw.json`. Keep the `.env` in the working directory as well, so the `latitude` CLI commands in steps 6 to 9 authenticate. `LATITUDE_PROJECT` and `LATITUDE_PROJECT_SLUG` name the same slug; use the spelling the harness documents.

**Verify auth before going further** (this catches the most common failure early):

```bash
latitude auth status     # expect:  ✓ active   LATITUDE_API_KEY env var
```

If it shows `missing`, `.env` isn't being applied: you're either not running from the app root (or a subdirectory), or an empty/stale `LATITUDE_API_KEY` in the shell is shadowing it (the `.env` load never overrides an already-set var). Fix the directory or `unset` the shadowing var, then re-check. See `latitude-cli` → Authentication.

**In hand:** `latitude auth status` shows the key active; no scratch file left.

### 4. Get consent and instrument (delegate to `latitude-telemetry`)

**App target.** Hand off to `latitude-telemetry` to add instrumentation, pointing it at `LATITUDE_PROJECT_SLUG=<projectSlug>`. The key/slug are already provisioned and in `.env`, so **skip that skill's MCP-config discovery detour**; you have the values. Follow its audit → group → clarify → **plan → wait for approval** → implement steps. Do not edit code before the user approves the plan. **Add the Artifact to that plan as its last item, before asking for approval**, so the user decides once and up front:

```text
- Finish: build your first Latitude Artifact, an HTML page showing what the telemetry captured from the verified session (model calls, tool calls, tokens, cost, timing, the conversation), saved to artifacts/first-session.html. Say "skip the artifact" to leave it out.
```

`go ahead` on the plan approves this item too.

**Harness target.** Use `latitude-telemetry`'s "Coding-agent / harness telemetry" entry instead of the app workflow: install the harness plugin the way its docs page says. There is no app code to plan, so the consent moment is the question you ask before installing: the harness's prompts, responses and tool I/O will be sent to Latitude, and the structural-only mode is available where one exists. If you already asked it together with the account question in Preflight, do not ask again. Put the Artifact in that same question as default scope, in one line, so it is decided here:

```text
I'll also build a first Artifact (artifacts/first-session.html) showing what Latitude captured from the verification run. Say "skip the artifact" to leave it out.
```

Do not ask a separate question for it.

`latitude-telemetry`'s workflow ends with its own "verify real traces land" step. In this orchestration that verification loop is steps 5 to 8 below, with step 7 adding the temporary-account cleanup, so drive the trace-checking from here rather than verifying twice.

**In hand:** instrumentation installed (or the harness plugin configured), the user's consent on record, and a note of whether they struck the Artifact.

### 5. Run the target's real LLM flow

**Do:** run the user's **actual** code so real spans are emitted, not a synthetic span. Spans typically export on a batch interval, so they may take a short while to arrive; poll rather than expecting them instantly (step 6). Let the process finish or shut down **gracefully** so buffered spans flush; a hard kill can drop them. For short-lived scripts, ensure the SDK flushes before exit (see `latitude-telemetry`).

**Harness target:** restart the harness so it loads the plugin, then run one real session through it (a short prompt that triggers at least one tool call, so the Artifact has a tool call to show) and let the session end normally so the plugin flushes.

**In hand:** one real run completed.

### 6. Inspect real traces and iterate

**Do:** poll the project's traces until the expected spans arrive, then check quality:

```bash
latitude traces list --project-slug <projectSlug> --format json
```

Confirm model, token counts, message capture, and span boundaries look right. If instrumentation is wrong (missing spans, no token data, wrong boundaries), fix it and re-run the target. **Loop until the traces are correct.**

**In hand:** traces that look right.

### 7. Clean the messy iteration traces, without touching config

The verification loop leaves noisy traces. **Do:** wipe them by deleting and recreating the project **with the same name**, which yields the **same slug**, so `LATITUDE_PROJECT_SLUG`, the harness config and all instrumentation stay valid and **no config is re-edited**:

```bash
latitude projects delete --project-slug <projectSlug>
latitude projects create --name "<the exact same project name from step 2>"
```

A successful `projects delete` returns HTTP 204 (no body), which the CLI renders as a placeholder like `{ bytes: 0, mimeType: text/plain, saved_file: download.txt, status: success }`; **that `status: success` is the delete succeeding**, not an error or a file download. Then run the target **once more** (as in step 5) to produce a single clean set of real traces.

**In hand:** one clean run in a clean project. That run is the session the Artifact will show.

### 8. Verify the clean run with the CLI and collect its data

**Do:** read the clean run back. This is the verification evidence and, at the same time, the data for the Artifact, so keep the JSON outputs:

```bash
latitude traces list      --project-slug <projectSlug> --format json                       # the trace row: ids, timing, tokens, cost, models, tags
latitude traces get       --project-slug <projectSlug> --trace-id <traceId> --format json  # adds the conversation of the last LLM span
latitude traces listSpans --project-slug <projectSlug> --trace-id <traceId> --format json  # every span: operation, model, tool name, timing, tokens
latitude traces getSpan   --project-slug <projectSlug> --trace-id <traceId> --span-id <spanId> --format json  # one span in full: messages, tool definitions, tool input/output
latitude traces getMemory --project-slug <projectSlug> --trace-id <traceId> --format json  # only if the target keeps long-term memory
```

Run `getSpan` for the tool-call span(s) and for one LLM span. Confirm the same things as step 6 on this clean trace: model, tokens, message capture (if content is on), tool call and result, sensible boundaries. Do not delete anything now.

**In hand:** the trace id, the span list, and the full span payloads of the clean session, saved as JSON.

### 9. Build the first Artifact from the template

The user has not opened Latitude yet. This page is where they first see what the telemetry actually captured, in a form they can open right now, so it is worth doing well. It is a single HTML file with the Latitude look, light and dark themes, and a JSON data blob you fill with the values from step 8. No script runs and no data is fetched: you paste what the CLI returned.

**Do:**

1. Get the template, `first-artifact.html`, bundled with this skill. If the skill is installed, copy it from the skill folder; otherwise fetch it:

   ```bash
   mkdir -p artifacts
   curl -fsSL https://raw.githubusercontent.com/latitude-dev/skills/main/skills/latitude-setup/first-artifact.html -o artifacts/first-session.html
   ```

2. Open `artifacts/first-session.html` and replace the example JSON inside `<script id="artifact-data" type="application/json">` with the real session. The comment above that tag maps each field to the CLI command that returns it. Keep the `id` and `type` attributes; change nothing else in the file. Rules:
   - Paste wire values as the CLI returns them: durations in **nanoseconds**, cost in **microcents**, times as ISO-8601. The page converts them.
   - `meta.project`, `meta.projectName`, `meta.projectUrl` (`https://console.latitude.so/projects/<slug>`), `meta.source` (the SDK or plugin and its version), `meta.contentCaptured` (`true` if prompts, responses and tool payloads are sent; `false` in structural-only mode).
   - `claim`: the claim link, its expiry and the email it was sent to, from step 2. **Temporary-account flow only**; set it to `null` on the existing-account path.
   - `trace`: the row from `traces list` / `traces get`, plus `url` = `https://console.latitude.so/projects/<slug>?tab=traces&traceId=<traceId>`.
   - `spans`: every span from `listSpans` with its `spanId`, `parentSpanId`, `name`, `operation`, `model`, `provider`, `toolName`, `startTime`, `endTime`, `statusCode`, tokens and cost fields.
   - `conversation`: `systemInstructions`, `inputMessages` and `outputMessages` from `traces get` or the LLM span's `getSpan`, pasted **exactly as returned** (the GenAI shape with `role` and `parts` is fine; so is `{ role, content }`). Include them only when `meta.contentCaptured` is `true`. If a message is very long, shorten the text inside its `content` field and keep the JSON valid; never cut a JSON string in the middle. **Never paste secrets, API keys or credentials that appear in a message; redact them.**
   - `tools.offered`: `toolDefinitions` (name and description) from the LLM span; `tools.calls`: name, input, output, duration and status from each tool span's `getSpan`, payloads only when content is captured.
   - `memory`: from `getMemory` when there are records; otherwise `null`.
   - `notes`: two to four sentences you write about this session: what the run did, what stands out in the numbers, what is not set yet (session id, user id, tags) and what setting it would unlock. Facts from the data, no filler.
3. **Render it before calling it done.** A parsing blob is not a working page. Open the file in a browser (headless is fine: `google-chrome --headless --dump-dom file:///…/first-session.html`, or a Playwright/Puppeteer one-liner) and confirm there is no console error, every section has content, and the claim button shows when `claim` is set. If a section reports "could not be rendered", fix that part of the blob.

**In hand:** `artifacts/first-session.html` with the real session, plus its absolute path.

If the template cannot be fetched (no network), write the page by hand with the same sections (at a glance, timeline, identity, usage and cost, conversation, tools, notes) and the claim button. Report a blocker only if you genuinely cannot read the clean trace; never silently drop the Artifact.

### 10. Hand everything back in one message

The final state: instrumented app or harness, one clean project of verified real traces, a working claim link, and the Artifact on the user's side, with the user never having touched the Latitude UI first. Send **one** message with all of the following. Every line is required; the last one has two forms:

```text
Latitude is set up.
- Configured: <what changed: SDK/plugin, files, profile>. <Restart requirement, if any.>
- Verified: <the real run that produced traces> landed in project "<name>" (<n> spans, <models/tools seen>).
- Claim your workspace: <claim link>. Expires <date, time UTC>; unclaimed temporary organizations are deleted then. <"Also emailed to <address>." if applicable.>   (temporary-account flow only)
- Your first Artifact: <absolute path>/artifacts/first-session.html. Open it in a browser: it shows <one sentence: what the session did and the headline numbers>, and has the claim button too.
    or, only if the user struck it:
- Want a first Artifact later? Ask: "build me an artifact of <question>" (docs: https://docs.latitude.so/more/artifacts.md).
```

Deliver the Artifact through the current surface: open it in the browser when you are on the user's machine and can; attach the file when the chat surface supports attachments; otherwise give the absolute path. Never publish it anywhere public; it may contain conversation content.

Do **not** print the API key. Do not wait for the user to claim the account before building or sending the Artifact. Do not split this into "claim link now, Artifact after you answer": that split is exactly how the Artifact gets dropped.

Before sending, check the message against the completion contract in Ground rules: configured, verified, claim link with expiry, restart note if owed, Artifact path with what it shows or the one-line offer. If any is missing, the run is not finished.
