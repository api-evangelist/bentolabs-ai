---
name: bentolabs-cli
description: Drives `bentolabs-cli`, the command-line client for Bento, from a terminal. Lists and inspects traces, trajectories, signals, issues, clusters, and analytics, and calls any Bento REST endpoint. Use when the user runs `bentolabs`, pipes Bento JSON into jq/grep, scripts Bento in CI, hits an endpoint with `bentolabs raw`, signs in with `auth login`, picks a workspace with `workspaces use`, refreshes the command list with `refresh`, chooses pretty/raw output, or debugs "no workspace selected" or `--workspace` errors. Not for instrumenting application code or adding tracing to an app (that is bentolabs-integrate), or porting from Raindrop or Langfuse (bentolabs-migrate).
metadata:
  version: "4.1"
---

# bentolabs-cli

`bentolabs-cli` drives [Bento](https://docs.bentolabs.ai) from a terminal: triage issues, read runs and signals, pull analytics, script Bento JSON into pipelines, or call any API endpoint. You sign in as **yourself** (`bentolabs auth login`), so the CLI has full dashboard parity.

**The one fact:** the command tree is generated from the Bento API. `bentolabs <group> <command> --help` is the authoritative, always-current signature; the `references/*.md` files are a routing snapshot. New endpoints appear after `bentolabs refresh`.

## What Bento records

Learn the model top-down, the way you'd navigate. An **issue** is the front door: a tracked problem Bento grouped across runs. Its evidence is **trajectories** (analyzed runs), the **findings** extracted from them, and the **signals** (learned detectors) that flag it. Underneath a trajectory is its raw **trace** (the dashboard calls it a **run**), made of **spans**. Look-alike findings form **clusters**.

Findings have no command group of their own. Pull them with `issues list-findings` or `trajectories list-findings`.

## Route a request to a resource

Match the request to a resource, then read that one `references/<resource>.md` (not the whole set). To find which resource owns a keyword: `grep -ril "<keyword>" references/`, or scan `commands.yaml` — the one-file machine-readable index of every command (method, path, positionals, query flags, body / workspace semantics) across all 17 groups.

| You want to… | Dashboard page | Read |
|---|---|---|
| triage a tracked problem and its evidence | Issues | `references/issues.md` |
| browse analyzed runs (suspicious / wins / errors) | Monitoring | `references/trajectories.md` |
| see raw recorded runs, one run's spans or body | Runs | `references/traces.md` |
| totals or a time series over runs (the charts) | Runs (charts) | `references/analytics.md` |
| see what's happening at a glance (scatter) | Monitoring (clusters) | `references/clusters.md` |
| inspect detectors and what each fires on | Signals | `references/signals.md` |
| train a new detector from labeled runs | Deep Search | `references/deep-search.md` |
| ask Bento's in-product AI assistant | Agent | `references/agent.md` |
| connect or check an external source (Langfuse) | Settings → Integrations | `references/credentials.md`, `references/sync.md` |
| manage workspaces and members | Settings → General / Members | `references/workspaces.md` |
| mint API keys (app / OTLP auth) | Settings → API keys | `references/api-keys.md` |
| set up alert delivery (Slack, email) | Settings → Notifications | `references/notification-channels.md` |
| tune finding extraction | Monitoring settings | `references/workspace-extraction-config.md` |

## Set it up (once per machine)

1. **Install** — run `scripts/install.sh` (needs Python 3.10+).
2. **Sign in** — run `scripts/sign-in.sh` (browser Allow/Deny; tokens go to the OS keychain).
3. **Pick a workspace** — run `scripts/pick-workspace.sh` to save a default, so you can drop `--workspace`.

## Run commands

The shape is always `bentolabs <group> <command> [POSITIONALS] [--flags]`. The per-resource files carry each command's invocation and what it does; these global rules live here only:

- **Positionals** are the path params, in order, always strings (e.g. `bentolabs signals get <signal-id>`).
- **`workspace_id` is never positional.** It's injected from `--workspace <id>` or the saved default. A few commands instead take it as a `--workspace-id <id>` query flag (where `--workspace` and the saved default don't apply); `--help` shows which.
- **Query params are `--flags`.** Repeat a flag for a list (`--tag a --tag b`); booleans are `--flag` / `--no-flag`.
- **Bodies** come from `--data '<json>'`, `--data-file <path>`, or piped stdin (`raw` ignores stdin).
- **Output**: `--output pretty` (default; colorized JSON for humans) | `raw` (compact JSON, one trailing newline). **As an agent, pass `--output raw` for your own reads** — it's cheaper to parse and avoids ANSI codes. Switch to `pretty` only when piping the result straight to the user.

```bash
bentolabs traces list --limit 5 --output raw
bentolabs traces list --output raw | jq -r '.items[].id' | head -20
```

## Before you mutate, confirm

Some commands change platform state — issues, credentials, members, settings. Before running any of these, **summarize the intended change and confirm with the user**, even if the user's earlier request implied it. A user who says "clean up the queue" does not mean "resolve all 50 open issues without asking."

Mutating commands include (non-exhaustive — `--help` will say if a command writes):

- `issues change-status`, `issues update`, `issues comment-on`, `issues create`
- `credentials create`, `credentials update`, `credentials delete`, `credentials run-health-check`
- `incidents acknowledge`, `incidents assign`, `incidents resolve`
- `signals create`, `signals update`, `signals delete`
- `notification-channels create`, `notification-channels update`, `notification-channels delete`, `notification-channels test`
- `api-keys create`, `api-keys revoke`
- `workspaces create`, `workspaces update`, `workspaces remove-member`
- `sync update-workspace-settings`
- `workspace-extraction-config put`
- any `bentolabs raw POST|PUT|PATCH|DELETE ...`

Reads (`list`, `get`, `list-*`, `analytics *`, `sync get-doctor`, `auth whoami`, `config show`) don't need confirmation.

## The raw escape hatch

When no generated command fits, call any path directly. You fill in `workspace_id` yourself; the body comes from `--data` / `--data-file` only.

```bash
bentolabs raw GET /health
bentolabs raw POST /workspaces/<ws-id>/issues --data '{"title":"..."}'
```

## Built-in commands

Hand-written, present even with no cached spec:

- `bentolabs auth login | whoami | logout` — sign in as yourself, check identity, sign out.
- `bentolabs workspaces use <id>` — save the default workspace.
- `bentolabs raw <METHOD> <PATH>` — ad-hoc request to any path.
- `bentolabs refresh` — re-pull the command list (picks up new endpoints).
- `bentolabs config show` · `config set api-base <url>` — inspect config; repoint for local dev.
- `bentolabs version` — installed CLI version.

## Worked example — "something's wrong in prod"

Start at the front door and drill into the evidence, in the order the issue shows it:

1. `bentolabs issues list --status open` — the triage queue.
2. `bentolabs issues get <issue-id>` — carries `linked_signal_ids` and evidence counts.
3. `bentolabs issues list-trajectories <issue-id>` — the analyzed runs behind it.
4. `bentolabs issues list-findings <issue-id>` — the observations on top.
5. `bentolabs signals get <signal-id>` — each linked detector.

When the CLI has no command for an action (for example, binding a signal to a notification channel as an alert rule is dashboard-only), say so and point at the dashboard or `bentolabs raw` — never invent a command.

## Reference and scripts

- `references/<resource>.md` — per-resource commands, when to use, and footguns. **Read the one you need**, confirm exact args with `--help`.
- `commands.yaml` — every command (across all groups) in one compact YAML manifest: method, path, positionals, query flag types, body / workspace semantics. Use it when you need to scan the whole surface (e.g. "which command takes a `credential_id`?") instead of grepping all the reference files. It's a snapshot — `--help` and `bentolabs refresh` remain authoritative.
- `troubleshooting.md` — failure modes and fixes ("no workspace selected", 401s, stale command list, the stdin/keychain gotchas).
- `scripts/` — **run** `install.sh`, `sign-in.sh`, `pick-workspace.sh`; `recipes.sh` has copy-paste pipelines.

## Related skills

- **`bentolabs-integrate`** — install the SDK and emit traces from application code.
- **`bentolabs-migrate`** — move a Raindrop or Langfuse setup to Bento.

The CLI talks *to* the platform from a terminal; those skills emit traces *from* your code.
