# Usage telemetry

The LiteLLM proxy can count how requests to it go, which providers it calls, and which Admin UI pages people open, then hand those counts to an HTTPS endpoint or keep them in its own database. It is **off unless you turn it on**, and nothing leaves your deployment unless you also set an endpoint

Telemetry never contains prompts, completions, API keys, user ids, team ids, model names, IP addresses or request bodies. Every field it can produce is listed on this page; a field that is not declared in the record schema cannot be exported

## Turn it on

Set a level and, if you want reports sent somewhere, an endpoint:

```bash
export LITELLM_TELEMETRY_LEVEL=basic   # off, basic or full
export LITELLM_TELEMETRY_ENDPOINT=https://telemetry.example.com/v1/reports   # optional
```

With an endpoint set, the proxy POSTs one JSON report per flush window to it. Without one, reports are kept in the `LiteLLM_TelemetryReport` table so an air-gapped install can export them by hand (see [Export reports](#export-reports)). Telemetry needs either an endpoint or a connected database to start, and an unknown level leaves it off

| Level | What is kept |
|---|---|
| `off` | Nothing. This is the default |
| `basic` | Instance info, request rows and provider attempt rows, without block counts, header keys or config keys |
| `full` | Everything in `basic`, plus block counts by type, allowlisted request header names, config key names and Admin UI events |

## What a report contains

Each worker folds what it sees into in-memory counters and emits one report per window (60 seconds by default). Reports are counts, sums and fixed-bucket histograms instead of individual requests

**Report header**

| Field | Meaning |
|---|---|
| `schema_version` | Version of the report format |
| `window_start`, `window_end` | Unix timestamps of the window |
| `instance.instance_id` | Random id stored in the proxy database, the same for every worker and across restarts. Without a database it is a new id on every boot |
| `instance.litellm_version` | The running LiteLLM version |
| `instance.telemetry_level` | The configured level |
| `instance.config_keys` | Names of allowlisted config keys that are set, never their values. Empty for now |
| `dropped_records` | Records that did not fit, since a window holds at most 2000 distinct rows |

**Request rows**, one per distinct combination of:

| Field | Meaning |
|---|---|
| `endpoint` | Route template, such as `/chat/completions`. Never the raw path |
| `provider` | Provider of the deployment that served the request, such as `anthropic`, or `null` when no provider was called |
| `deployment_hash` | First 16 hex characters of SHA-256 over the instance id and the deployment id, so it cannot be matched across installs |
| `litellm_status` | Status class the client got: `2xx`, `3xx`, `4xx`, `5xx` |
| `provider_status` | Status class of the last provider call, or `none` |
| `litellm_cache_hit` | Whether the LiteLLM response cache answered |
| `provider_cache_hit` | Whether the last provider call reported a prompt cache read |
| `stream` | Whether the response was streamed |

Each request row carries `request_count`, sums of `input_tokens`, `output_tokens`, `cache_read_tokens` and `cache_write_tokens`, and histograms of `latency_total_ms`, `latency_to_headers_ms` and `latency_to_first_token_ms` as seen by the client, plus `provider_attempts` (how many provider calls retries and fallbacks made). At `full` it also carries a `block_count` histogram, `block_types` counts (`text`, `image`, `audio`, `file`, `tool_use`, `tool_result`, `thinking`, `other`) and `header_keys` counts

`header_keys` only names headers from a fixed allowlist: `anthropic-beta`, `anthropic-version`, `openai-beta`, `openai-organization`, `x-litellm-api-key`, `x-litellm-disable-callbacks`, `x-litellm-enable-message-redaction`, `x-litellm-num-retries`, `x-litellm-tags`, `x-litellm-timeout`, `x-stainless-lang` and `x-stainless-package-version`. Any other header is counted as `other`. Header values are never read

**Provider attempt rows** count every individual provider call, keyed on `provider`, `deployment_hash`, `provider_status` and `stream`, with an `attempt_count`, a `latency_ms` histogram and a `latency_to_first_token_ms` histogram. A request that was refused by the proxy before routing, for example a bad key or an exceeded budget, adds no attempt row

**Admin UI events** (`full` only) count page views and tab clicks as `page`, `action` and `target`, for example `{"page": "playground", "action": "click", "target": "tab=compare", "count": 1}`. The page is the first segment of the dashboard route and never contains an id, and the proxy rejects any event that does not match a short lowercase pattern. The browser sends these to the proxy's own `POST /telemetry/ui_events` route, never to a third party

Latency histograms use the bucket bounds 50, 100, 250, 500, 1000, 2500, 5000, 10000, 30000, 60000 and 120000 ms, with a last bucket for anything slower

## Export reports

When reports are kept locally, a proxy admin or admin viewer can page through them:

```bash
curl -s "http://localhost:4000/telemetry/reports?limit=500" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

The response holds `reports` plus `next_after` and `next_after_id`. Pass those back as `after` and `after_id` to get the next page, until `next_after` is `null`. Reports older than `LITELLM_TELEMETRY_RETENTION_DAYS` (30 by default) are deleted, and windows with no traffic are not stored

## Settings

| Environment variable | Description |
|---|---|
| `LITELLM_TELEMETRY_LEVEL` | `off`, `basic` or `full`. Default is `off` |
| `LITELLM_TELEMETRY_ENDPOINT` | HTTPS URL that receives one JSON report per window. When unset, reports go to the local table |
| `LITELLM_TELEMETRY_FLUSH_INTERVAL_SECONDS` | Length of a report window. Default is `60` |
| `LITELLM_TELEMETRY_SETTLE_TIMEOUT_SECONDS` | How long a finished request waits for its provider attempts to be logged before its row is written. Default is `2` |
| `LITELLM_TELEMETRY_RETENTION_DAYS` | How long locally kept reports are retained. Default is `30` |

The old `--telemetry` CLI flag still parses but does nothing
