---
name: bentolabs-integrate
description: Use when greenfield-integrating Bento into an app with no existing AI observability SDK. Presents three paths in order of preference — direct export (point a framework's own OpenTelemetry exporter at Bento; no Bento SDK in TypeScript, while Python frameworks attach the Bento JSON span processor), the one-line Google ADK auto-instrument (`bentolabs-sdk[adk]` + `bento.instrument()`), and manual per-call `bento.track_ai`. Triggers include "send my LangChain / Pydantic AI / Mastra / Vercel AI SDK traces to Bento", "point my exporter at Bento", wiring the Google ADK integration (`bento.instrument()`), manually tracking LLM calls with `bento.track_ai`, registering identity getters at `bento.init`, grouping multi-step agent flows with `bento.begin` trajectories, mapping OpenTelemetry GenAI / OpenInference semantic conventions to Bento dashboard columns, and debugging missing traces or empty dashboard columns. Covers Python SDK install, the `bl_pk_` API key, the four must-pass arguments to `track_ai` (`user_id`, `convo_id`, `model`, `provider`), input/output capture, properties type fidelity, the `flush()` / `shutdown()` lifecycle, and the lower-level OTel transport for apps with an existing TracerProvider. If the project already uses Raindrop or Langfuse, use the `bentolabs-migrate` skill instead.
metadata:
  version: "3.1"
---

# Bento

Bento is production infrastructure for AI agents. You send it traces; its dashboard turns them into traces you can read, signals (plain-English failure-mode detectors), alerts, evaluations, and versioned improvements. Under the hood Bento is an OpenTelemetry collector, so it understands standard `gen_ai.*` and `openinference.*` span attributes. The Python SDK is the only generally-available SDK today; the TypeScript SDK is still in development.

There are three ways to get traces into Bento. They are listed best-first.

1. **Direct export — point a framework's own exporter at Bento.** If the app already uses a framework with its own OpenTelemetry exporter, you send its traces to Bento by setting an endpoint and a `Bearer` header. One catch: Bento accepts **JSON only** (not protobuf yet). In **TypeScript** that's easy and needs no Bento package (Vercel AI SDK, Mastra). In **Python**, stock exporters send protobuf, so you attach the Bento JSON span processor plus an OpenInference instrumentor (LangChain, LlamaIndex, Pydantic AI). This is the best choice when you control the framework.
2. **Google ADK auto-instrument — one line.** If the app uses Google ADK, you install `bentolabs-sdk[adk]`, call `bento.instrument()` once at startup, and every model call, tool call, and agent step is captured automatically. No per-call code.
3. **Manual tracking — one call per LLM site.** For raw LLM SDKs (OpenAI, Anthropic, Bedrock, Vertex) that have no native exporter, you add a `bento.track_ai(...)` call next to each LLM call.

The two SDK paths (2 and 3) work together: a manual `track_ai` call and the spans ADK captures inside the same `bento.begin(...)` block end up in one trace.

## The plan

Copy this checklist into your reply and tick the boxes as you go.

```
Bento integration progress:
- [ ] Step 1: Look at the codebase and figure out which path fits.
- [ ] Step 2: Get an API key. Install the Bento SDK only if your path needs it.
- [ ] Step 3: Wire up ONE path: direct export, ADK auto-instrument, or manual track_ai.
- [ ] Step 4: Make sure each trace carries a user id and a conversation id.
- [ ] Step 5: Run a real flow, check the dashboard, fix any empty columns.
```

Do these in order. Do not skip Step 1 — what you find there decides everything after it.

## Step 1: Look at the codebase

Run the helper script from the top folder of the project:

```bash
./scripts/discover.sh
```

It only reads files. It prints labelled sections: the language and Python version, the web framework, every LLM call site, any agent framework that might have a native exporter (section 1c2), Google ADK usage (section 1d), existing OpenTelemetry setup, where env vars live (section 1f), and any competing SDK like Raindrop or Langfuse (section 1g).

Read every section. Three things in the output decide what you do next:

- **Which path to use in Step 3.** If section 1c2 found an agent SDK with a native exporter, lean toward direct export. If section 1d found Google ADK, the ADK auto-instrument path fits. Otherwise it's manual `track_ai`.
- **Where to put the API key.** Section 1f shows the env file.
- **Whether there's already a tracing SDK.** Section 1g flags Raindrop or Langfuse — handle that next.

If this is not a Python project, the Bento SDK paths (ADK and manual) do not apply, because the Python SDK is the only GA one today. But **TypeScript apps can use direct export today with no Bento SDK** — you point the framework's own JSON exporter at Bento. So if the app uses a framework with a native exporter (Vercel AI SDK, Mastra), go straight to `references/PATH-DIRECT-EXPORT.md`. Otherwise point the user at `https://docs.bentolabs.ai/typescript` for SDK status.

### If Raindrop or Langfuse is already installed

If section 1g found Raindrop or Langfuse, this might be a migration, not a greenfield install. Do not guess — ask the user which they want:

- **Migrate** off the old SDK to Bento. If they pick this, stop here and switch to the `bentolabs-migrate` skill. That skill owns the rename guides, the instrumentor setup, and the safe "keep both running until verified" workflow. Do not try to migrate from this skill.
- **Keep the old SDK** and add Bento next to it (for an A/B comparison, or to instrument only new code). If they pick this, continue with Step 3 below, and mention in your summary that two SDKs will be sending spans.

### Summarize what you found before editing anything

Write a short summary back to the user: the language and Python version, the web framework, whether ADK is in use, the list of LLM call sites (with file and line and provider), whether OpenTelemetry is already set up, where the env vars live, and whether Raindrop or Langfuse is present (and which path the user picked). Confirm with them before you start editing.

## Step 2: Get an API key (and install only if you need to)

Get a key from `https://platform.bentolabs.ai`. It starts with `bl_pk_`. The SDK checks the prefix immediately and raises `BentoAuthError("invalid_api_key_format")` on a bad key before it makes any network call.

Whether you install the Bento SDK depends on your path:

- **Direct export installs nothing from Bento in TypeScript.** You only need the key in the environment (`BENTOLABS_API_KEY`) and, if you're not using the default `https://api.bentolabs.ai`, `BENTOLABS_BASE_URL`. (Python direct export attaches the Bento span processor, so it does need the SDK — install it as below.)
- **ADK and manual need the SDK.** Run `./scripts/install.sh`. It picks the package manager the project already uses (uv, poetry, pdm, or pip), installs `bentolabs-sdk`, and adds a `BENTOLABS_API_KEY` placeholder to `.env`. For ADK, set `ADK_PRESENT=1` first so it adds the `[adk]` extra.

A note on `bento.init()`: in the plain `track_ai` flow it's optional — the first `track_ai` call sets itself up from the env vars. But for the ADK path, call `bento.init()` yourself at startup, so your identity getters (Step 4) are registered before the first span is captured.

## Step 3: Wire up ONE path

Pick the first path that fits what Step 1 found, and follow its reference file.

1. **Direct export.** Use this if section 1c2 found a framework with a native OpenTelemetry exporter (LangChain, Pydantic AI, Mastra, Vercel AI SDK, and others). You point that exporter at Bento. In TypeScript this needs no Bento package; in Python you attach the Bento JSON span processor (Bento accepts JSON only). Open `references/PATH-DIRECT-EXPORT.md`, find your framework in the table, and copy its snippet.
2. **ADK auto-instrument.** Use this if section 1d found Google ADK. It's three lines at startup and zero per-call code. Open `references/PATH-A-ADK.md`.
3. **Manual `track_ai`.** Use this for raw LLM SDKs with no native exporter, or for any call site the first two paths don't cover. You add one `track_ai` call per LLM call. Open `references/PATH-B-MANUAL.md`.

You can combine the ADK and manual paths in one app. When you open a `bento.begin(...)` trajectory, the spans ADK captures inside it and your own `track_ai` and `tool_span` calls all end up in the same trace.

## Step 4: Make sure each trace has a user id and a conversation id

Two ids unlock the most useful dashboard features: `user_id` (the user filter) and `convo_id` (the conversation timeline). Make sure every trace carries them.

- **ADK and manual paths:** read `references/IDENTITY.md`. There are two ways to supply the ids. The first is to register getter functions once at `bento.init(...)` — preferred for the ADK path, and it works for the manual path too. The second is to pass the ids as keyword arguments on each `track_ai` call — manual path only. Pick one and use it consistently. The same reference also covers setting the ids late, after a trace has already started (`bento.update_current_trace` and `bento.propagate_attributes`).
- **Direct-export path (TypeScript):** there is no Bento SDK, so you don't use `bento.init` getters. Instead the ids are two plain span attributes — `gen_ai.user.id` and `gen_ai.conversation.id` — that you set through your own framework. `references/PATH-DIRECT-EXPORT.md` explains where. (Python direct export uses the Bento span processor, so you can register `bento.init` getters as above.)

## Step 5: Check it works

First, run the smoke test for your path:

- **Manual path:** `python scripts/verify-manual.py`. It sends a `hello_world` event and flushes; a row should appear in the dashboard within seconds.
- **ADK path:** `python scripts/verify-integration.py`. It prints `activated: 'adk'` when the integration is on. Then run one real ADK agent call and confirm a span shows up.
- **Direct-export path:** there's no script and no Bento worker to check — just run one real flow and look at the dashboard (the loop below).

If you're on an SDK path and want to confirm the background worker that ships spans is running, run `python scripts/check-worker.py`. The thread list it prints should include `OtelBatchSpanRecordProcessor`. If it doesn't, `init()` failed quietly or your code called the SDK before `init()` finished — re-check Step 2.

### Then check every call site

For each place you instrumented in Step 3:

1. Run the user flow once so that call site actually fires.
2. Open `https://platform.bentolabs.ai` and find the new row.
3. Check all six columns are filled: `provider`, `model`, `input`, `output`, `user_id`, `convo_id`.
4. If any column is empty, go back to Step 3 for that call site, fix it, and run again.
5. Repeat until every column is filled.

If `user_id` or `convo_id` is empty on ADK-captured spans, your init-time getter returned `None`. Walk the checklist in `references/TROUBLESHOOTING.md`. Don't move on to other work until every site passes.

## Things that are easy to get wrong

Read these before you write any instrumentation. They are the mistakes that happen most often.

- **You must pass `provider=` yourself.** Bento does not guess it from the model name, and if you skip it the provider filter and grouping break. Always pass it: `provider="openai"`, `"anthropic"`, `"aws_bedrock"`, `"google"`, and so on.
- **A Bedrock model needs `provider="aws_bedrock"`, even when the model id starts with `anthropic.`** For example `anthropic.claude-3-sonnet-20240229-v1:0` is `provider="aws_bedrock"`. The model id is intentionally ambiguous.
- **`convo_id` must be the same string for every turn of one conversation.** If you mint a new UUID per request, each turn becomes its own row and the conversation timeline falls apart. The usual bug is generating the id inside the request handler instead of reading it from the path/route.
- **`track_ai` and `begin` use `convo_id=`. `init`, `update_current_trace`, and `propagate_attributes` use `session_id=`.** Same value, different keyword name depending on the function — the single most common naming mistake. Passing `session_id=` to `begin` doesn't get ignored; it raises a `TypeError`.
- **Call `bento.init()` and `bento.instrument()` once at startup, never per request.** They're safe to call twice, but running a partial `init()` on every request churns the identity registration.
- **Background threads don't inherit the open trajectory.** Threads and `concurrent.futures` workers do not carry the trajectory `ContextVar`. Wrap the work you submit with `contextvars.copy_context().run(...)` so it inherits. `asyncio` tasks inherit on their own.
- **Short-lived programs can lose the last batch.** Scripts, notebooks, Lambdas, and anything that calls `os._exit` may exit before the last batch ships. Call `bento.flush()` before you exit. Long-running services flush automatically on a clean shutdown, but a hard kill skips that.
- **`bento.instrument()` returning `None` means the `[adk]` extra isn't installed in the active virtualenv.** It does not raise — it just returns `None`. Re-run `pip install "bentolabs-sdk[adk]"` in the same venv and confirm with `pip show bentolabs-sdk`.
- **`track_ai` and `begin` detach from the surrounding OpenTelemetry context on purpose.** Don't "fix" this by reattaching them — the part of Bento that reads the spans depends on the detachment.

Lower-frequency footguns (such as how `properties=` values keep or lose their type) live in `references/REFERENCE.md` and `references/TROUBLESHOOTING.md`. Read those before writing instrumentation.

## Reference

For the direct-export path (per-SDK support matrix and config), read `references/PATH-DIRECT-EXPORT.md`.

For deep reference (public surface, the four kwargs that must be on every call, trajectories, properties, lifecycle, the lower-level OTel transport for apps with an existing `TracerProvider`), read `references/REFERENCE.md`.

For diagnostics when traces don't appear or columns are empty, read `references/TROUBLESHOOTING.md`.

For deeper docs hosted at `docs.bentolabs.ai` as plain Markdown, see the URL list in `references/DOCS-INDEX.md`.

## Related

- For migrating from an existing SDK (Raindrop or Langfuse) instead of greenfield, use the `bentolabs-migrate` skill.
- For driving Bento from a terminal after the integration, use the `bentolabs-cli` skill.
