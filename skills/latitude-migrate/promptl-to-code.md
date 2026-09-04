# PromptL to code

Conversion rules for moving a Latitude V1 PromptL document into application code. Keep the original `.promptl` in `latitude-v1-export/prompts/` until the converted prompt is verified against real traffic.

## The mapping

| PromptL | In code | Notes |
| --- | --- | --- |
| Frontmatter `provider`, `model` | The provider SDK you instantiate and the `model` argument | One SDK per provider; the telemetry instrumentation must receive that same module |
| Frontmatter `temperature`, `maxTokens`, `topP`, `stopSequences`, and friends | Provider-call arguments | Check each against the installed SDK version. Anthropic Python 1.x has no `temperature` on `messages.create()`; drop or move such keys and note it in the plan |
| Frontmatter `schema` (JSON output) | The provider's structured-output feature, or a JSON instruction plus parsing | Keep the parser the app already had for `response.text` |
| Frontmatter `tools` | Provider tool definitions plus your own tool-call loop | Tool calls show up in Latitude's Tools view automatically |
| Frontmatter `type: agent`, `maxSteps` | Your agent loop or framework | The loop is application code now; `maxSteps` becomes your own guard |
| Frontmatter `agents` (subagents) | Function calls or your framework's delegation | Give each subagent its own `capture()` name inside the parent boundary |
| `<system>` block | The `system` argument (Anthropic) or the first `system` message (OpenAI) | |
| `<user>` / `<assistant>` blocks | Messages in the provider's message array | Preserve order |
| `{{ variable }}` | String formatting or a template function | Every undefined variable in V1 was an input parameter; make it a function argument |
| `{{#if}}` / `{{else}}` | Language conditionals when building the messages | |
| `{{#each}}` | Loops when building the messages | |
| Snippets (`<prompt path="...">`) | Shared constants or functions | |
| `<step>` chains | Sequential provider calls inside one `capture()` | Each step is a span; the last chat span is what Latitude renders as the conversation |
| Comments `/* ... */` | Delete, or move rationale to the commit | |

## Version strings

V1 tracked prompt versions as drafts and a published version pinned by `versionUuid`. In V2 the version is a string you own, sent as `metadata.prompt_version` on every trace. Bump it when the prompt changes, and put the app release in a `release-<version>` tag so Experiments can compare cohorts.

```python
PROMPTS = {"2.0.0": ("grammar-check@2", GRAMMAR_CHECK_V2), "2.1.0": ("grammar-check@3", GRAMMAR_CHECK_V3)}
```

## Example

V1 document `grammar-check.promptl`:

```markdown
---
provider: anthropic
model: claude-3-5-sonnet-latest
temperature: 0
---
<system>
You are a grammar coach. The learner is studying {{ language }}.
Reply ONLY with JSON: {"corrections": [...], "is_correct": true|false}
</system>

<user>
Language: {{ language }}
Text: {{ text }}
</user>
```

Python, called directly and wrapped for telemetry:

```python
GRAMMAR_CHECK_VERSION = "grammar-check@2"
GRAMMAR_CHECK_SYSTEM = """You are a grammar coach. The learner is studying {language}.
Reply ONLY with JSON: {{"corrections": [...], "is_correct": true|false}}"""

def check_grammar(text: str, language: str, *, user_id: str, session_id: str) -> str:
    return capture(
        "grammar-check",
        lambda: client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=600,
            system=GRAMMAR_CHECK_SYSTEM.format(language=language),
            messages=[{"role": "user", "content": f"Language: {language}\nText: {text}"}],
        ).content[0].text,
        {
            "user_id": user_id,
            "session_id": session_id,
            "tags": ["grammar", f"release-{APP_RELEASE}", "production"],
            "metadata": {"prompt_version": GRAMMAR_CHECK_VERSION, "release": APP_RELEASE},
        },
    )
```

TypeScript, same prompt:

```ts
const GRAMMAR_CHECK_VERSION = "grammar-check@2"
const system = (language: string) =>
  `You are a grammar coach. The learner is studying ${language}.\nReply ONLY with JSON: {"corrections": [...], "is_correct": true|false}`

export async function checkGrammar(text: string, language: string, ctx: { userId: string; sessionId: string }) {
  return capture(
    "grammar-check",
    async () => {
      const res = await client.messages.create({
        model: "claude-haiku-4-5-20251001",
        max_tokens: 600,
        system: system(language),
        messages: [{ role: "user", content: `Language: ${language}\nText: ${text}` }],
      })
      return res.content[0].type === "text" ? res.content[0].text : ""
    },
    {
      userId: ctx.userId,
      sessionId: ctx.sessionId,
      tags: ["grammar", `release-${APP_RELEASE}`, "production"],
      metadata: { prompt_version: GRAMMAR_CHECK_VERSION, release: APP_RELEASE },
    },
  )
}
```

## The gateway call it replaces

```python
result = await sdk.prompts.run("grammar-check", RunPromptOptions(parameters={"text": text, "language": language}))
data = json.loads(result.response.text)
```

Map `parameters` to the function arguments, `customIdentifier` to `session_id` or `user_id`, `result.response.text` to the provider's content accessor, and V1 streaming callbacks (`onEvent`, `onFinished`) to the provider's streaming API consumed inside the `capture()` callback.


## `type: agent` with tools: a full example

V1 prompt (OpenAI, an agent that classifies a support message and looks up a policy page before answering):

```
---
provider: OpenAI
model: gpt-4.1-mini
type: agent
tools:
  - latitude/extract
---
<system>
Classify the request, use latitude/extract to consult https://example.com/policy
when rules matter, then answer. Return ONLY JSON: {"category": "...", "response": "..."}
</system>
<user>
{{username}}
{{customer_query}}
</user>
```

`latitude/extract` has no V2 equivalent, so it becomes a real fetch:

```js
export const policyLookupTool = {
  type: "function",
  function: { name: "lookup_policy", description: "Fetches the policy page as plain text.",
              parameters: { type: "object", properties: {}, required: [] } },
}

export async function runPolicyLookup() {
  const res = await fetch("https://example.com/policy", { signal: AbortSignal.timeout(5000) })
  const html = await res.text()
  return html.replace(/<[^>]+>/g, " ").replace(/\s+/g, " ").trim().slice(0, 4000)
}
```

And the gateway call (`prompts.run` / `prompts.chat`) becomes a two-pass loop, one `capture()` per turn:

```js
async function runTurn(messages, onDelta) {
  const first = await client.chat.completions.create({
    model: MODEL, messages, tools: [policyLookupTool], tool_choice: "auto", stream: true,
  })
  let buffered = ""
  const calls = {}
  for await (const chunk of first) {
    const delta = chunk.choices[0]?.delta
    if (delta?.content) { buffered += delta.content; onDelta(delta.content) }
    for (const tc of delta?.tool_calls ?? []) {
      const slot = (calls[tc.index] ??= { id: "", name: "", arguments: "" })
      if (tc.id) slot.id = tc.id
      if (tc.function?.name) slot.name += tc.function.name
      if (tc.function?.arguments) slot.arguments += tc.function.arguments
    }
  }
  const call = Object.values(calls)[0] // the prompt says "never call it more than once"
  if (!call) { messages.push({ role: "assistant", content: buffered }); return }

  messages.push({ role: "assistant", content: null,
    tool_calls: [{ id: call.id, type: "function", function: { name: call.name, arguments: call.arguments } }] })
  messages.push({ role: "tool", tool_call_id: call.id, content: await runPolicyLookup() })

  const second = await client.chat.completions.create({ model: MODEL, messages, stream: true })
  let finalText = ""
  for await (const chunk of second) {
    const piece = chunk.choices[0]?.delta?.content
    if (piece) { finalText += piece; onDelta(piece) }
  }
  messages.push({ role: "assistant", content: finalText })
}
```

`onDelta` is the caller's SSE (or equivalent) writer, so streaming to the client keeps working exactly as it did against the V1 gateway. `capture()` wraps one call to `runTurn`, so both passes land as one trace with `system`, `user`, `assistant`, `tool`, `assistant` messages in order.
