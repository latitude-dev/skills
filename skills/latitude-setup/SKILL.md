---
name: latitude-setup
description: Zero-account onboarding orchestrator for Latitude. Bootstrap a temporary Latitude account from the terminal (no signup), instrument the app or agent harness (Claude Code, Hermes, OpenClaw, Pi, Prime Intellect) for tracing, verify real traces, clean up, hand back a browser link to claim ownership, and build the first Artifact (HTML report) from the new traces in the same handoff. Use when someone wants to set up Latitude, install Latitude telemetry, or "try Latitude" and may have no account or API key yet — the landing-page "try it with your agent" flow. If it turns out the user already has an account, API key, or connected Latitude MCP, this skill redirects to latitude-telemetry + latitude-cli instead of creating a temporary account.
---

# Latitude Setup (zero-account onboarding)

Orchestrates the **from-scratch** path: the user has **no Latitude account and no API key**. This skill provisions a temporary account via the CLI, instruments the target, verifies real traces, and returns a claim link the user opens in a browser to take ownership.

The **target** is whatever should emit traces: an **app** (instrumented with the Latitude SDKs or an OTLP exporter) or an **agent harness** the user runs locally (Claude Code, Hermes, OpenClaw, Pi, Prime Intellect, each with its own plugin). Everything below applies to both; the steps call out where a harness differs.

This skill depends on three others — **`latitude-cli`** (install + auth + command primitives), **`latitude-telemetry`** (instrumentation) and **`latitude-artifacts`** (the first report, step 9). Read them; this skill only adds the orchestration between them.

## Preflight: don't create a temporary account if the user already has one

The temporary-account bootstrap exists to make onboarding automatic for someone with **nothing set up yet**. But a user who already has a Latitude account can land on this skill by mistake — and routing them through a redundant temporary org plus a claim step is worth avoiding when you can. So before bootstrapping anything, infer from the environment whether an account already exists, and **redirect instead of bootstrapping** if it clearly does:

- **An API key is already present** — `LATITUDE_API_KEY` set in the shell, in a `.env` (search the app root and its parents), in the app's secret manager, or in deployment/CI config.
- **A Latitude MCP is connected and authenticated** in this harness — an OAuth-authorized Latitude MCP means the user already has a workspace.
- **Existing Latitude config/instrumentation in the repo** — a `LATITUDE_PROJECT_SLUG`, a `@latitude-data/telemetry` / `latitude-telemetry` dependency, or an OTLP exporter already pointed at `ingest.latitude.so`.
- **A harness already wired to Latitude** — `LATITUDE_API_KEY` in `~/.hermes/.env`, a `LATITUDE_*` env block in `~/.claude/settings.json`, `~/.pi/agent/latitude-telemetry.json`, or a `diagnostics.otel` block in `~/.openclaw/openclaw.json` pointing at `ingest.latitude.so`.
- **The user says so** — they mention being signed in, having a project, or already using Latitude.

**If any of these hold → do NOT bootstrap.** The user has an account; redirect to the direct path:

1. Use **`latitude-telemetry`** via its "invoked directly" entry point to audit and instrument the app against the existing key/project. Use **`latitude-cli`** if you still need to install/authenticate the CLI or discover the project slug (`latitude projects list`).
2. Finish with `latitude-telemetry`'s verification step: run the user's **real** LLM flow and confirm the traces landed via the Latitude **MCP, CLI, or API**. Do **not** run this skill's bootstrap, claim-link, or delete-and-recreate cleanup — those belong only to the temporary-account flow, and the cleanup would destroy a project the user actually owns.
3. Once traces are verified, build the first Artifact exactly as in step 9 below. The completion contract applies to this path too.

**If genuinely nothing is set up** (no key, no MCP, no existing config) → proceed with the temporary-account flow below.

**If you can't confidently tell either way → default to bootstrapping; don't stall on it.** This detection is best-effort. A bootstrapped temporary organization is low-stakes and recoverable: the claim link lets it be redeemed later, and it can be claimed even into an account the user already has — so creating one for a user who turned out to already have an account is a minor, fixable outcome, not a failure. The overriding goal is to get the app instrumented and emitting real traces to Latitude **autonomously**; don't block that on perfect account detection. If the user is right there and a one-line question is cheap, you may confirm first — but absent a clear signal, bootstrap and keep going.

## Ground rules

- **Completion contract.** Setup is **not done when traces arrive**, and not done when the claim link is delivered. It is done when the final handoff (step 9) contains, in one message: what was configured, the verified real trace, the claim link and its expiry, any restart the user still owes, **and the first Artifact** (its path and top finding), or, only if the user struck the Artifact from the plan, a one-line offer to build it later. An ending without one of those two is an incomplete run, the same as ending before verification. Do not treat "traces verified" or "claim link sent" as the finish line.
- **The Artifact is decided in the plan, not at the end.** The plan the user approves in step 4 lists the Artifact as its last item, on by default. Approval of the plan is the go-ahead to build it; there is no second question at the end, and nothing to forget.
- **Plan, then wait.** Instrumentation goes through `latitude-telemetry`'s "present a plan, wait for explicit approval" contract. Do not edit app code before approval.
- **Never print raw secrets.** The bootstrap API key must never appear in chat, logs, or commits. The claim link is safe to show (it's the whole point) — the API key is not.
- **One project, named once.** Bootstrap creates exactly one project. There is no throwaway "testing" project; cleanup is delete + recreate with the same name (see step 7).

## The flow

### 1. Install dependencies

Ensure the `latitude-telemetry`, `latitude-cli` and `latitude-artifacts` skills are available (`npx skills add https://github.com/latitude-dev/skills --skill latitude-telemetry,latitude-cli,latitude-artifacts` installs any that are missing; if that fails, continue, and step 9 explains what to do), and install the `latitude` binary as described in **`latitude-cli` → Install** (OS/ARCH detection, download the matching release asset, place on `PATH`, `chmod +x`). **Only `cli-5.0.0` and later are the real Latitude CLI** — `latitude-cli` explains why earlier `cli-*` tags must be ignored. Confirm with `latitude --version` (expect ≥ 5.0.0).

### 2. Bootstrap a temporary account

This is unauthenticated — no key needed yet. Infer a sensible project name from the app or harness (for a harness, its name works: "hermes", "claude-code") and optionally an org name; ask the user for an email only if you want the claim link mailed to them.

**First, discover the command's exact flags and response fields** — don't assume the shape; it can change across versions:

```bash
latitude account bootstrap --schema
```

**Keep the API key out of your own conversation.** `--format json` prints the API key to stdout, which lands in the transcript. On a best-effort basis, capture the response to a scratch file instead of letting it print, then display everything *except* the secret:

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

- the **API key** — an org-scoped secret; never print it. It flows straight into `.env` in step 3, then the scratch file is deleted.
- the **project slug** — the single created project; your telemetry target and the cleanup handle (step 7).
- the **claim link** (plus its expiry, and the email it was sent to if you passed one) — handed back to the user in step 8.

### 3. Configure auth via `.env`

Write the key and project slug into the app's `.env` **straight from the scratch file, so the value never prints**. Make sure `.env` is gitignored. Replace any existing `LATITUDE_*` lines rather than duplicating them, then delete the scratch file:

```bash
touch .env
# Drop any prior LATITUDE_API_KEY / LATITUDE_PROJECT_SLUG lines (avoids duplicates). BSD+GNU sed:
sed -i.bak '/^LATITUDE_API_KEY=/d;/^LATITUDE_PROJECT_SLUG=/d' .env && rm -f .env.bak
# Append the real values from the file — never echoed (field names .apiKey/.projectSlug per --schema):
printf 'LATITUDE_API_KEY=%s\n'     "$(jq -r .apiKey      ~/.latitude-bootstrap.json)" >> .env
printf 'LATITUDE_PROJECT_SLUG=%s\n' "$(jq -r .projectSlug ~/.latitude-bootstrap.json)" >> .env
# Remove the scratch file so the key isn't left on disk:
rm -f ~/.latitude-bootstrap.json
```

If any value you later add to `.env` contains spaces (not the key/slug — e.g. a header/token value), wrap it in double quotes: the CLI's `.env` parser rejects unquoted spaced values and stops, after which it won't read `LATITUDE_API_KEY` (see `latitude-cli` → Authentication).

`LATITUDE_API_KEY` authenticates **both** the `latitude` CLI (it auto-loads `.env` from the working directory and its parents) **and** the telemetry SDK — one entry, both consumers. **Do not** run `latitude auth login`; it invokes the OS keychain and can block on a prompt. Run subsequent `latitude` commands from the app root so `.env` is picked up. Do not echo the key back to the user.

**Harness target:** the harness does not read the app's `.env`. Copy the two values (never echoed, same `jq` pattern) to wherever that harness reads them, per its docs page: Hermes takes `LATITUDE_API_KEY` and `LATITUDE_PROJECT` in `~/.hermes/.env`; Claude Code and Pi take them as installer flags (`--api-key`, `--project`); OpenClaw takes them as headers in `~/.openclaw/openclaw.json`. Keep the `.env` in the working directory as well, so the `latitude` CLI commands in steps 6 and 7 authenticate. `LATITUDE_PROJECT` and `LATITUDE_PROJECT_SLUG` name the same slug; use the spelling the harness documents.

**Verify auth before going further** (this catches the most common failure early):

```bash
latitude auth status     # expect:  ✓ active   LATITUDE_API_KEY env var
```

If it shows `missing`, `.env` isn't being applied — you're either not running from the app root (or a subdirectory), or an empty/stale `LATITUDE_API_KEY` in the shell is shadowing it (the `.env` load never overrides an already-set var). Fix the directory or `unset` the shadowing var, then re-check. See `latitude-cli` → Authentication.

### 4. Instrument the app (delegate to `latitude-telemetry`)

Hand off to `latitude-telemetry` to add instrumentation, pointing it at `LATITUDE_PROJECT_SLUG=<projectSlug>`. The key/slug are already provisioned and in `.env`, so **skip that skill's MCP-config discovery detour** — you have the values. Follow its audit → group → clarify → **plan → wait for approval** → implement steps. Do not edit code before the user approves the plan.

**Add the Artifact to that plan as its last item, before asking for approval**, so the user decides once and up front:

```text
- Finish: build your first Latitude Artifact, a self-contained HTML baseline report of the traces this setup produces (volume, latency, cost, tokens, tool calls), saved to artifacts/overview.html. Say "skip the artifact" to leave it out.
```

`go ahead` on the plan approves this item too. If the user strikes it, note that and keep going; step 9 then ends with an offer instead of a report.

**Harness target:** use that skill's "Coding-agent / harness telemetry" entry instead of the app workflow: install the harness plugin the way its docs page says, ask before installing (the harness's prompts, responses and tool I/O will be sent to Latitude) and offer the structural-only mode where one exists. There is no app code to plan; the approval is for the plugin install, the config edits, and the same Artifact line item.

`latitude-telemetry`'s workflow ends with its own "verify real traces land" step. In this orchestration that verification loop is steps 5–6 below — and step 7 then extends it with the temporary-account cleanup — so drive the trace-checking from here rather than verifying twice.

### 5. Run the user's real LLM flow

After instrumentation, run the user's **actual** code so real spans are emitted — not a synthetic span. Spans typically export on a batch interval, so they may take a short while to arrive — poll rather than expecting them instantly (step 6). Let the process finish or shut down **gracefully** so buffered spans flush; a hard kill can drop them. For short-lived scripts, ensure the SDK flushes before exit (see `latitude-telemetry`).

**Harness target:** restart the harness so it loads the plugin, then run one real session through it (a short prompt that triggers at least one tool call) and let the session end normally so the plugin flushes.

### 6. Inspect real traces and iterate

Poll the project's traces until the expected spans arrive, then verify quality:

```bash
latitude traces list --project-slug <projectSlug> --format json
```

Confirm model, token counts, message capture, and span boundaries look right. If instrumentation is wrong (missing spans, no token data, wrong boundaries), fix it and re-run the user's code. **Loop until the traces are correct.**

### 7. Clean the messy iteration traces — without touching app config

The verification loop leaves noisy traces. Wipe them by deleting and recreating the project **with the same name**, which yields the **same slug** — so `LATITUDE_PROJECT_SLUG` and all instrumentation stay valid and **no config is re-edited**:

```bash
latitude projects delete --project-slug <projectSlug>
latitude projects create --name "<the exact same project name from step 2>"
```

A successful `projects delete` returns HTTP 204 (no body), which the CLI renders as a placeholder like `{ bytes: 0, mimeType: text/plain, saved_file: download.txt, status: success }` — **that `status: success` is the delete succeeding**, not an error or a file download. Then run the user's code **once more** to produce a single clean set of real traces. Confirm with `latitude traces list` again.

### 8. Prepare the claim link

The claim link is safe to show (it's the whole point); the API key is not. Have the link, its expiry and, if you passed an email, the note that it was also mailed, ready for the handoff in step 9. Do not send them yet: the claim link and the Artifact go out together.

### 9. Build the first Artifact and hand everything back in one message

Telemetry is live, so the same agent can now *read* it. Delegate to the **`latitude-artifacts`** skill via its "delegated from `latitude-setup`" entry point. The approved plan is the go-ahead: auth and the project slug are already in `.env`, so the skill skips discovery and its intake and builds its *first-artifact default* (a reliability, latency and cost overview over the traces so far, saved to `artifacts/overview.html`) without asking anything. The project is minutes old, so the report is a baseline, not a trend, and the skill says so in the page.

If `latitude-artifacts` is not installed, install it now (`npx skills add https://github.com/latitude-dev/skills --skill latitude-artifacts`). If that fails (no network, no `npx`), read the skill straight from the repo or the docs page (<https://docs.latitude.so/more/artifacts.md>) and build a simpler single-file report by hand from a few `latitude analytics query` calls. Report a blocker only if you genuinely cannot read the project's data; never silently drop the Artifact.

Then send **one** final message with all of the following. Every line is required; the last one has two forms:

```text
Latitude is set up.
- Configured: <what changed: SDK/plugin, files, profile>. <Restart requirement, if any.>
- Verified: <the real run that produced traces> landed in project "<name>" (<n> traces, <models/tools seen>).
- Claim your workspace: <claim link>. Expires <date, time UTC>; unclaimed temporary organizations are deleted then. <"Also emailed to <address>." if applicable.>   (temporary-account flow only; omit on the existing-account path)
- Your first Artifact: artifacts/overview.html (open it in a browser). Top finding: <one sentence from the report>.
    or, only if the user struck it from the plan:
- Want a first Artifact later? Ask: "build me an artifact of <question>" (docs: https://docs.latitude.so/more/artifacts.md).
```

Do **not** print the API key. Do not wait for the user to claim the account before building or sending the Artifact. Do not split this into "claim link now, Artifact after you answer": that split is exactly how the Artifact gets dropped.

Before sending, check the message against the completion contract in Ground rules: configured, verified, claim link with expiry, restart note if owed, Artifact path with top finding or the one-line offer. If any is missing, the run is not finished.
