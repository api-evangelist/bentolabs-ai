---
name: bentolabs-migrate
description: Use when moving an existing AI observability or agent stack to Bento. Covers two cases — an old analytics SDK (Raindrop, Langfuse), or an agent SDK (Mastra, OpenAI Agents SDK, LangChain/LangGraph, LlamaIndex, Vercel AI SDK, CrewAI, Pydantic AI) whose traces currently go to Arize, Phoenix, Langfuse, or LangSmith. Triggers include "migrate from Raindrop", "migrate from Langfuse", "replace raindrop-ai", "replace langfuse", "send our <SDK> traces to Bento", "point our OpenTelemetry exporter at Bento", and existing usage of `@observe`, `from langfuse.openai`, `raindrop.track_ai`, `langfuse.score`, `ArizeExporter`, or `OTLPSpanExporter`. Presents the three migration paths in priority order — Path A direct OTLP export (JSON only; no Bento SDK in TypeScript, Python uses the span processor), Path B OpenInference instrumentor wired to a `BentoLabsSpanProcessor`, Path C manual per-call `track_ai`/`begin` — plus the `convo_id` vs `session_id` kwarg footgun, the now-required `provider=`, and the safe coexistence window where both stacks run until verification passes. For a greenfield app with no SDK at all, use `bentolabs-integrate`.
metadata:
  version: "2.0"
---

# Bento migration

Use this skill when something already sends traces somewhere, and you want those traces to go to Bento instead. There are two common situations:

- **An old analytics SDK is installed** — Raindrop (`raindrop-ai`) or Langfuse (`from langfuse`, `@observe`).
- **An agent SDK is sending traces to another tool** — Mastra, OpenAI Agents SDK, LangChain/LangGraph, LlamaIndex, Vercel AI SDK, CrewAI, or Pydantic AI, currently pointed at Arize, Phoenix, Langfuse, or LangSmith.

One fact makes the whole migration simpler: **Bento is just an OpenTelemetry collector.** It accepts standard OpenTelemetry traces over HTTP at `${BENTOLABS_BASE_URL}/v1/traces`, using the header `Authorization: Bearer bl_pk_...`, and it reads the standard `gen_ai.*` and `openinference.*` attributes. One constraint to know up front: **Bento accepts JSON payloads only** (gzip is fine); it rejects protobuf for now. That is why the first migration path can avoid installing the Bento SDK entirely — but only in TypeScript, where exporters can send JSON. In Python, stock OpenTelemetry exporters send protobuf, so Python frameworks take Path B instead (details in Step 3).

If the project has no tracing at all, this is the wrong skill. Use `bentolabs-integrate` instead.

## The plan

Copy this checklist into your reply and tick the boxes as you go.

```
Bento migration progress:
- [ ] Step 1: Find out what the project uses today.
- [ ] Step 2: Get a Bento API key. Only install the Bento SDK if Step 3 needs it.
- [ ] Step 3: Pick ONE path: A (direct export), B (instrumentor), or C (manual).
- [ ] Step 4: Wire up the path you picked.
- [ ] Step 5: Run one real flow, check the dashboard, THEN remove the old tool.
```

Important rule for the whole migration: **do not remove the old SDK or old exporter until Step 5 passes.** If you remove it first, there is a window where nothing is recording. Keep both running, get Bento working, confirm it, and only then take the old one out.

## Step 1: Find out what the project uses today

Run the helper script from the top folder of the project:

```bash
./scripts/detect.sh
```

It only reads files; it changes nothing. It prints several sections: the old tracing SDK (if any), the agent SDK in use and whether it has its own exporter, where traces currently go, and any raw LLM clients.

Read every section. Then write a short summary back to the user — what you found, and which path you think fits (Step 3 explains the paths). Before you edit anything, confirm the user actually wants to move to Bento. Some teams keep two tools running on purpose to compare them, so ask if that's the case.

## Step 2: Get a Bento API key

Every path needs a key. Get one from `https://platform.bentolabs.ai`. It starts with `bl_pk_`. Put it in the environment as `BENTOLABS_API_KEY`.

Whether you install the Bento SDK depends on the path you pick in Step 3:

- **Path A (direct export) installs nothing from Bento — in TypeScript.** You only need the key and the base URL. (Python frameworks use Path B, which does need the SDK.)
- **Path B and Path C need the SDK.** Run `./scripts/install-bento.sh` to add it. If the app uses Google ADK, set `ADK_PRESENT=1` first so the script adds the `[adk]` extra. The script keeps the old SDK installed, which is what you want.

If you are not sure which path you'll use yet, do Step 3 first and come back.

## Step 3: Pick ONE path

There are three ways to get traces into Bento. They are listed best-first. Read each one, decide if it fits, and **stop at the first one that matches.** Most agent-SDK migrations end on Path A. Most Raindrop/Langfuse migrations end on Path B for the auto-captured calls, plus Path C for the handful of custom ones.

Before you start: **if the app uses Google ADK, do not migrate it here.** There is a simpler one-line option (`bentolabs-sdk[adk]`) in the `bentolabs-integrate` skill. Use that for ADK and only come back here for anything ADK doesn't cover.

**Path A — direct export (try this first, TypeScript).** This fits when the framework has its own OpenTelemetry exporter that can point at any URL, send a `Bearer` header, and emit **JSON** (Bento rejects protobuf). In **TypeScript** that's the common case — you set the endpoint and auth header and install nothing from Bento (the Vercel AI SDK and Mastra both work this way). In **Python** this path isn't available yet, because stock Python exporters only send protobuf; Python frameworks use Path B instead. Open `references/NATIVE-EXPORT.md` for the per-framework table, copy-paste snippets, and the JSON-vs-protobuf details.

**Path B — OpenInference instrumentor.** This fits when there is no native exporter, but there is an `openinference-instrumentation-<sdk>` package for your LLM SDK. This is the path for Raindrop's `auto_instrument=True` and for Langfuse's drop-in clients (`from langfuse.openai`, `from langfuse.langchain`). You install the instrumentor and register it on a `BentoLabsSpanProcessor`; your LLM call sites stay exactly as they are. Install the instrumentors with `./scripts/install-instrumentors.sh openai anthropic ...` and read `references/PATHS.md` for the wiring.

**Path C — manual translation.** This is the last resort, for everything the first two paths don't cover: custom decorators, hand-written `track_ai` calls, bespoke spans. You rename each call site to `bento.track_ai` or `bento.begin` by hand. The exact renames for each old SDK are in `references/RAINDROP.md` and `references/LANGFUSE.md`.

## Step 4: Wire up the path you picked

- **Path A:** Follow the snippet for your SDK in `references/NATIVE-EXPORT.md`. Set the endpoint to `${BENTOLABS_BASE_URL}/v1/traces` and the header to `Authorization: Bearer ${BENTOLABS_API_KEY}`. Leave the old exporter in place for now.
- **Path B:** Register each instrumentor on a `BentoLabsSpanProcessor`, following `references/PATHS.md`. If you are coming from Langfuse, also swap `from langfuse.openai import OpenAI` back to the plain `from openai import OpenAI` — the instrumentor wraps the plain client.
- **Path C:** Open the guide for your old SDK — `references/RAINDROP.md` or `references/LANGFUSE.md` — and walk it call site by call site.

## Step 5: Check it works, then remove the old tool

First, prove the connection works at all. How you do this depends on your path:

- **Path B or Path C** (the Bento SDK is installed): run `python scripts/verify.py`. It sends one event and flushes it, and a row should show up in the dashboard within a few seconds.
- **Path A** (direct export, no Bento SDK): there is no script to run — `verify.py` imports the Bento SDK, which you did not install on this path. Instead, just run one real flow of your app and check the dashboard directly, as described next.

Next, prove your real code works. For every call site you changed (and, on Path A, every kind of flow), run the user flow once and open the dashboard. Check that the new row has all six of these columns filled in: `provider`, `model`, `input`, `output`, `user_id`, `convo_id`. If any column is empty, go back to Step 4 and fix it.

Only after every column is filled, remove the old tool:

- Raindrop: `pip uninstall raindrop-ai`, and delete the `RAINDROP_*` env vars.
- Langfuse: `pip uninstall langfuse`, and delete the `LANGFUSE_*` env vars.
- A repointed exporter (Path A): delete the old exporter's config.

The exact per-SDK commands are also in `references/RAINDROP.md` and `references/LANGFUSE.md`.

## Things that are easy to get wrong

Read these before you start. They are the mistakes that happen most often.

- **Don't remove the old tool early.** Again: keep it until Step 5 passes. Otherwise nothing records in the gap.
- **`track_ai` and `begin` use `convo_id=`. `init`, `update_current_trace`, and `propagate_attributes` use `session_id=`.** It is the same value (the conversation/session id) but the keyword name is different depending on the function. This is the single biggest mistake when coming from Langfuse, where everything was `session_id`. If a `track_ai` or `begin` call lands without a session in the dashboard, check the keyword name first. Note: passing `session_id=` to `begin` is not just ignored — it raises a `TypeError`.
- **You must pass `provider=` yourself now.** Bento does not guess the provider from the model name. Always pass it, e.g. `provider="openai"`. One catch: a Bedrock model id like `anthropic.claude-3-sonnet-...` needs `provider="aws_bedrock"`, not `"anthropic"`.
- **On Path A, the exporter must send JSON and an `Authorization: Bearer` header.** Bento rejects protobuf (you'll get a 415), and some exporters default to protobuf or to a vendor-specific header. The classic example is Mastra's `ArizeExporter`, which may send protobuf and switches to Arize's own headers if `ARIZE_SPACE_ID` is set. `references/NATIVE-EXPORT.md` shows the JSON-compatible config (e.g. Mastra's `OtelExporter` with `protocol: "http/json"`).

The mistakes that are specific to one old SDK are in `references/RAINDROP.md` and `references/LANGFUSE.md`. Read the one that matches your project.

## Where the details live

- `references/NATIVE-EXPORT.md` — Path A: the per-SDK table and copy-paste config.
- `references/PATHS.md` — Path B and Path C setup, with code.
- `references/RAINDROP.md` and `references/LANGFUSE.md` — the line-by-line rename guides for each old SDK.
- `references/TROUBLESHOOTING.md` — what to do when traces don't show up or columns are empty.

The full online guides are at `https://docs.bentolabs.ai/migrations/raindrop.md` and `https://docs.bentolabs.ai/migrations/langfuse.md`.

## Related skills

- Starting from nothing (no tracing SDK at all)? Use `bentolabs-integrate`.
- Want to read traces from the terminal after migrating? Use `bentolabs-cli`.
