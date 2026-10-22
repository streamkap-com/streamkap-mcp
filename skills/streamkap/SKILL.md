---
name: streamkap
description: Manage Streamkap CDC pipelines, API sources, destinations, transforms, topics, and infrastructure via CLI or MCP. Use when the user needs to create, monitor, troubleshoot, or manage data streaming pipelines and connectors.
---

# Streamkap CLI & MCP

## Setup

**CLI:** `npm install -g @streamkap/tools` then `streamkap <command>`
**MCP:** `npx -y @streamkap/tools` (stdio) or remote HTTP at `mcp.streamkap.com/mcp`

**Auth — pick one:**
- `STREAMKAP_CLIENT_ID` + `STREAMKAP_CLIENT_SECRET` (Settings > API Keys). REST tools only.
- `STREAMKAP_PROJECT_KEY` (Settings > Project Keys, then `streamkap auth encode-key <file>`). Bundles API + Kafka + Schema Registry + tool scoping. HTTP mode: use `X-Streamkap-Project-Key` header. Individual vars win on conflict. See https://docs.streamkap.com/project-keys.

## Command Pattern
streamkap <resource> <action> [id] [--flags] --json

Always use `--json` for structured output. Do not parse text output.

## MCP Response Formats
MCP tools accept `response_format` parameter:
- `"json"` (default): Compact JSON — best for programmatic processing
- `"text"`: Readable plain-text summary — ~34% fewer tokens, better for LLM reasoning

Use `response_format: "text"` for overview/diagnostic tasks. Use `"json"` when you need to extract specific fields.

## CLI Output Formats
- `--json` or `--format json`: Machine-readable JSON (default when piped)
- `--format text`: Human-readable summary (default in terminal)
- `--format auto`: JSON when piped, text when TTY (the default)

CLI JSON output is never truncated; CLI text output truncates at 480,000 characters. The MCP server caps tool responses at the same ~480K limit and emits a structured `truncated` envelope when a response would exceed it (see `## Pagination` below).

Other global flags: `--body <file>` (or `--body -` for stdin) supplies the request body as JSON for create/update/bulk commands — explicit flags override its fields. `--dry-run` previews a destructive command without executing it. `--verbose` logs method/path/status/timing to stderr.

## Pagination
All list endpoints use **1-indexed pages** with defaults: `page=1`, `page_size=10` (max 100).

Responses include `page` and `page_size`. Some endpoints return `total` (total item count); others return `has_next` (boolean) instead — these are alternatives, never both.

```bash
streamkap pipelines list --page 1 --page-size 50 --json   # First 50
streamkap pipelines list --page 2 --page-size 50 --json   # Next 50
```

To get all results, increase `--page-size` (up to 100), or iterate pages using `total` to compute page count or `has_next` to know when to stop.

## Capabilities
- CRUD pipelines, sources, destinations, transforms, tags
- **Brief tools**: `streamkap_list_sources_brief`, `streamkap_list_destinations_brief`, `streamkap_list_pipelines_brief`, `streamkap_list_transforms_brief` — fast overview with minimal data, ideal for initial discovery
- Monitor: metrics, logs, organisation statistics, consumer groups
- Lifecycle: deploy CDC sources; pause, resume, restart, stop, reset sources and destinations (reset re-reads an API source's history); `sources retry-api` recovers a Broken API source
- Snapshots (CDC sources only): execute, stop, cancel (initial and incremental)
- Schema registry: list subjects, get schemas by version/ID
- Auth: manage access tokens, client credentials, API keys
- Alerts: manage subscribers, preferences, credentials
- Usage: usage metrics, usage summary, usage export
- Cluster Scaling: check scaling status, scale clusters, monitor/cancel operations
- Kafka: manage kafka users, list topics, read topic messages, produce messages with optional Schema Registry encoding (Avro/JSON/Protobuf), consume messages with optional Schema Registry decoding (optional — requires KAFKA_BOOTSTRAP_SERVERS)
- Bulk operations: bulk-stop, bulk-resume, bulk-reset, bulk-delete, bulk-snapshot (CDC only); bulk pipeline delete skips an API source's managed pipelines
- **Workflow prompts**: troubleshoot-pipeline, setup-pipeline, infrastructure-overview, diagnose-dlq-errors, report-pipeline-health, deploy-monitoring-agent, triage-agent-error

## Tool Profiles
Tool profiles restrict which tools are available. Set via `MCP_TOOL_PROFILE` env var, `X-Streamkap-Tool-Profile` header, or a Project Key (`tool_profile`, `allowed_tools`, `blocked_tools` fields — server-verified in both stdio and HTTP modes):
- `full` (default): all tools
- `read-only`: GET/search tools only — safe for monitoring agents
- `agent-operator`: read + idempotent writes (pause, resume, restart); no create or delete
- `infra-admin`: everything except direct Kafka data tools
- `directory-default`: reads + plain & idempotent writes + Kafka data tools; blocks destructive resets and deletes

Toolsets narrow the catalog by feature group before profile and allow/block rules. Set `MCP_TOOLSETS=sources,topics`, send `X-Streamkap-Toolsets` in HTTP mode, or add `?toolsets=` to the MCP URL. Restrictions from env, header, URL, and the server-verified Project Key are intersected; unknown names and disjoint intersections fail closed. Available groups: `pipelines`, `sources`, `destinations`, `topics`, `transforms`, `observability`, `tags`, `schema-registry`, `consumer-groups`, `admin`, `kafka-access`, `alerts`, `usage`, `cluster`, `tenant`, `kafka`, `meta`, `workflows` (`diagnose_pipeline`, `health_overview`), `docs` (`search_docs`, `get_docs_page`, `list_docs_pages`, `get_docs_navigation`).

OAuth sessions use the compact catalog by default; `MCP_CATALOG=compact` selects it elsewhere. It keeps a curated core plus four meta-tools in `tools/list`; HTTP clients can request the same narrowing with `X-Streamkap-Catalog: compact` or `?catalog=compact`. In compact mode:

1. Use `streamkap_search_tools` to find an authorized long-tail capability.
2. Use `streamkap_describe_tool` to get its exact input schema.
3. Call it through `streamkap_invoke_read_tool` or, only when the active profile permits it, `streamkap_invoke_write_tool`.

The dispatchers validate the target schema and reapply the same profile, toolset, allow, and block checks. Compact mode does not grant access to hidden tools.

If toolsets are also configured, include `meta` (for example, `MCP_TOOLSETS=sources,meta`) or the discovery and dispatch tools are intentionally hidden.

## Limitations
- Real-time event streaming: MCP stdio is request-response only — no way to keep a Kafka consumer alive between tool calls. For real-time streaming use the remote HTTP server or the CLI `streamkap kafka subscribe` command. `streamkap_consume_messages` works in all modes for one-shot inspection.
- No undo for destructive operations (deletes are permanent)
- Rate limits apply — 429 errors include retry-after info
- All operations are scoped to one tenant (the authenticated API key)
- MCP tool responses are capped at ~480KB. Oversized responses are replaced with a structured envelope (`{ ..., truncated: { total_records, returned_records, hint } }`) — when you see `truncated.truncated=true`, paginate, lower `page_size`/`max_messages`, use `response_format: "text"`, or call `*_brief` tools

## Destructive Commands (modify or delete data)
These commands change live infrastructure. When stdin is a terminal (a human is
present) they prompt for a y/N confirmation; `--yes` skips the prompt. When stdin
is not a terminal (agent/script/CI), they execute without confirmation.
Use `--dry-run` to preview first:
- `delete`, `bulk-delete` — permanently remove resources
- `pause`, `stop` — halt data flow
- `reset` — clear consumer group offsets (data loss risk); on an API source it re-reads its history
- `execute-snapshot`, `execute-incremental-snapshot` — trigger re-sync
- `deploy` — activates pending CDC resources; API sources reconcile automatically

## Resource Groups
pipelines | sources | destinations | transforms | topics | tags |
dashboard | schema-registry | consumer-groups | admin |
kafka-access | alerts | usage | cluster-scaling | tenant | other

Helper commands (not API resources):
auth (login/logout/status/token/encode-key) | doctor | completions | version | api-spec | agent (set/clear/status) | skill (install) | kafka (produce/consume/subscribe with --schema-registry, --decode, --pattern, --group-id)

## Common Workflows

### API sources

HubSpot, Salesforce, NetSuite, Stripe, Zendesk, Google Analytics 4, Facebook Ads and Google Ads use the generic source commands.

- Config comes from `sources get-source-connector-schema <code>`: fields, `auth_mode` choices, `capabilities` and `oauth`. Everything goes in `config`, including `resources` (required, at least one) and the optional `backfill_start`.
- Every vendor except Zendesk works with credentials the customer creates; Zendesk signs in only through Connect.
- `available: false` with `unavailable_reason` means no new source of that connector here; create answers 403.
- `sources validate-source-connection <code>` tests credentials and resources without saving (pass `source_id` on an edit). A refused credential or setting names its config field in `field`; each resource carries `needs` and `fix` when a permission is missing.
- `sources update`: resend the mask or omit a secret to keep it, `null` clears an optional one. A new Zendesk `subdomain` needs a new grant; a new Salesforce `domain` or NetSuite `account_id` needs the credentials sent again.
- Connect: `sources start-source-oauth-connect <code>` (`client_id`/`client_secret` via `--body` where the schema lists `oauth.client_fields`), open `authorize_url`, then `sources poll-source-oauth-grant <code> --state <state>`. Put the one-time `oauth_grant_id` in `config` within 10 minutes and keep it out of logs.
- API sources reconcile by themselves; `sources deploy` and snapshots are CDC only. `sources get-api-source-resources-status <source-id>` shows backfill progress per resource; `sources retry-api <source-id>` recovers a Broken one (400 for a CDC source, 409 if it is not Broken).
- Discovery on an existing source: `sources list-source-available-objects <source-id>` (custom objects to add to `resources`) and `sources list-source-available-properties <source-id>` (properties to narrow to). `sources list-source-available-accounts <code>` lists the accounts new credentials reach.
- Route topics with `topics attach-topic-destination <topic-id> <destination-id>`, or several of one source at once with `destinations attach-topics-to <destination-id> --topic-ids <id>,<id>`; `topics list-topic-destinations <topic-id>` shows where a topic goes. Topic delivery is for API sources only.
- An API source has no pipeline of its own: delivery keeps one managed pipeline per source and destination (`managed: true`, its `binding_id`). `pipelines update`/`delete` answer 409 on it and `pipelines create` from an API source answers 400; change delivery through the topic-destination commands.

### Workflow tools (MCP only — prefer these over hand-assembling primitives)

- `streamkap_health_overview` — tenant-wide health in one call: broken
  connectors by name, throttled sources by name, DLQ topics holding
  messages, stale sources, lag and latency, plus per-status counts. Start here.
- `streamkap_diagnose_pipeline(pipeline_id)` — one pipeline in one call: source
  and destination status, the failed task's Kafka Connect trace, throughput
  metrics, recent ERROR logs, and rule-based findings. Replaces six to eight
  metric and log calls.
- `streamkap_search_docs(query)` — turns a connector error string into the
  documentation page that explains it.
- `streamkap_get_docs_page(route)` — reads that page in full, as Markdown.
  `streamkap_list_docs_pages` and `streamkap_get_docs_navigation` enumerate the
  docs when a search is too narrow.

`Throttled` is an API source (and its managed delivery pipelines) waiting out its vendor
rate budget. It is neither healthy nor broken, and it resumes by itself at the
`resume_at` in its task status (`suspend_reason: rate_budget`). Do not resume,
restart or retry it: that clears the wait and spends the vendor quota again.
`Throttled` is also a valid `status` filter on source and pipeline list/search.

### Check infrastructure health (start here)
streamkap dashboard stats --json
# MCP: streamkap_get_organisation_statistics (response_format: "text")

### List and inspect pipelines
streamkap pipelines list-brief --json                     # Quick overview (use first)
streamkap pipelines list --page-size 100 --json           # Full list (up to 100)
streamkap pipelines get <id> --json
streamkap pipelines metrics <id> --json
# MCP: use list_pipelines_brief first, then get_pipeline for details

### Manage sources
streamkap sources list-brief --json
streamkap sources get <id> --json
streamkap sources pause <id> --yes --json
streamkap sources resume <id> --yes --json
streamkap sources metrics <id> --json

### Troubleshoot
streamkap dashboard logs --query-string "error" --json
streamkap sources metrics <id> --json
streamkap destinations metrics <id> --json
streamkap topics messages --topic-id <id> --json          # Sample data records

### Bulk operations
streamkap sources bulk-stop --ids '["id1","id2"]' --yes --json
streamkap destinations bulk-resume --ids '["id1","id2"]' --yes --json

### Switch organisation / tenant (OAuth only)
# When the user belongs to multiple Streamkap orgs and asks to operate on a non-default one.
# Only available via the OAuth (Bearer) connector flow — not exposed when authed via API key
# or Project Key (those credentials are already tenant-scoped).
# MCP: streamkap_get_tenant_trees                           # discover available tenants
# MCP: streamkap_switch_tenant({tenant_id: "..."})          # switch, then re-run any read tool
# MCP: streamkap_get_organisation_statistics                # confirm context
# Note: Frontegg's switch updates the user's GLOBAL active tenant — concurrent dashboard
# or MCP sessions for the same user will flip on their next token refresh.

## Error Handling
Exit 0=success, 1=general error, 2=bad args (or a declined confirmation), 3=auth,
4=permission denied, 5=not found, 6=conflict, 7=rate limit, 8=server error,
9=timeout, 10=config error, 11=network error.
With --json, errors include "code", "message", "suggestion" fields.

## Tips
- Use `list-brief` variants for quick overviews with less data
- Use `--page` and `--page-size` for pagination (page starts at 1, max page_size is 100)
- Use `--dry-run` before destructive commands to preview what will happen
- Use `--verbose` to see HTTP method/path/status/timing on stderr
- Pipe to jq: `streamkap pipelines list --json | jq '.result[]'`
- JSON params (--config, --tags, --ids) accept JSON strings: `--config '{"key":"value"}'`
- For large payloads, prefer `--body`: `streamkap sources create --body ./source.json --json`
