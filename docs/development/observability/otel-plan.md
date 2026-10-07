# Agent observability plan: OpenTelemetry → CloudWatch / AgentCore and Datadog

Status: proposal (nothing below is implemented yet). Companion reference:
[claude-agent-sdk-observability.md](claude-agent-sdk-observability.md) (verbatim copy of the
Agent SDK observability page).

## TL;DR

- **The Agent SDK does not produce telemetry of its own.** It spawns the same Claude Code CLI as
  a child process and passes OTel env vars through. Every span the SDK "emits"
  (`claude_code.interaction`, `claude_code.llm_request`, `claude_code.tool`, …) is emitted by the
  CLI. Our `claude -p` driver can produce **identical** spans today without switching harness.
- `claude -p` also honors an inbound `TRACEPARENT` / `TRACESTATE` (interactive sessions ignore it;
  `-p` and the SDK both honor it). So parent-linking the CLI under our own orchestrator span also
  works on the CLI path — the SDK only automates the injection.
- What is actually missing today:
  1. No OTel configuration anywhere in the runtime (`lambda/agentcore`, `terraform/modules/compute/agentcore`).
  2. CloudWatch's OTLP endpoints (X-Ray traces, CloudWatch Logs) require **SigV4** auth; the CLI's
     OTLP exporter only supports static headers. → need a collector in the container to sign.
  3. No parent span in the orchestrator (`http-server.js` / `commands/run-stage.js`), so each CLI
     run would be a disconnected root trace.
  4. Datadog has the same "where do I send OTLP" question, and the same collector answers it.
- Recommendation: **Phase 1** (OTel on the existing CLI path + in-container collector fanning out to
  CloudWatch and Datadog) gets ~all the value. **Phase 2** (optional `claude-sdk` driver) is about
  programmatic control (hooks, typed messages, permission callbacks), not about getting spans out.

## Where the CLI is invoked today

| Path | File | Spawn helper |
| - | - | - |
| Stage run + resume | `lambda/agentcore/commands/run-stage.js` | `runChild` / `captureChild` |
| Conflict resolution | `lambda/agentcore/commands/resolve-conflict.js` | `runChild` |
| One-shot LLM calls (derive, enrichment) | `lambda/agentcore/cli/one-shot.js` | `captureChild` |
| Capabilities probe | `lambda/agentcore/commands/capabilities.js` | `captureChild` |
| argv/env construction | `lambda/agentcore/cli/drivers.js` (`claudeDriver`) | — |
| Env merge | `lambda/agentcore/cli/spawn.js` (`{ ...process.env, ...env }`) | — |

Claude Code is pinned at `2.1.246` in `lambda/agentcore/Dockerfile`. The driver sets
`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`; per the env-var docs that governs Anthropic-bound
traffic (auto-update, Anthropic usage metrics, error reporting). OTel export to *your* collector is
a separate opt-in (`CLAUDE_CODE_ENABLE_TELEMETRY=1`). Verify in Phase 1, step 1.

## Phase 1 — OTel on the existing `claude -p` path (recommended first)

### 1. Prove the CLI exports from the container (half a day)

Add to the Claude driver env (or the runtime env in Terraform) and point at a local collector:

```bash
CLAUDE_CODE_ENABLE_TELEMETRY=1
CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1          # traces are beta and need this
OTEL_TRACES_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
OTEL_TRACES_EXPORT_INTERVAL=1000               # short-lived process: flush fast
OTEL_LOGS_EXPORT_INTERVAL=1000
OTEL_METRIC_EXPORT_INTERVAL=5000
OTEL_SERVICE_NAME=aidlc-stage-executor
CLAUDE_CODE_OTEL_DIAG_STDERR=1                 # surface exporter errors on stderr (>= 2.1.179)
```

Never use the `console` exporter: stdout is the `stream-json` channel. `runChild` already tees a
stderr tail when `captureStderrTail` is set, so exporter diagnostics land in the container log.

### 2. Run an OTel Collector inside the runtime container

AgentCore Runtime is one container per microVM, so there is no sidecar; run the collector as a
background process started before `http-server.js` (small entrypoint script, or spawned and
supervised by `http-server.js`). The microVM persists per `runtimeSessionId`, so the collector
survives across `/invocations` calls in a session.

- Binary: `otelcol-contrib` (arm64) pinned + sha256-verified, same pattern as bun/uv in the
  Dockerfile. Contrib has `sigv4auth`, `awsemf`, and `datadog` in one distro. (The ADOT collector
  is fine too if you only target AWS.)
- Credentials: the default AWS chain inside AgentCore resolves to the runtime execution role.

Sketch (`lambda/agentcore/otel/collector.yaml`):

```yaml
extensions:
  sigv4auth/xray: { region: ${env:AWS_REGION}, service: xray }
  sigv4auth/logs: { region: ${env:AWS_REGION}, service: logs }

receivers:
  otlp:
    protocols:
      http: { endpoint: 127.0.0.1:4318 }

processors:
  batch: { timeout: 1s }
  resource:
    attributes:
      - { key: deployment.environment, value: ${env:DEPLOY_ENV}, action: upsert }
      - { key: aws.log.group.names, value: ${env:AGENT_LOG_GROUP}, action: upsert }
  # Claude Code already sets gen_ai.system / gen_ai.request.model / gen_ai.response.id on
  # claude_code.llm_request. Copy the token counts to GenAI semconv names so GenAI-aware
  # backends (AgentCore GenAI Observability, Datadog LLM Observability) pick them up.
  transform/genai:
    trace_statements:
      - context: span
        statements:
          - set(attributes["gen_ai.usage.input_tokens"], attributes["input_tokens"]) where name == "claude_code.llm_request"
          - set(attributes["gen_ai.usage.output_tokens"], attributes["output_tokens"]) where name == "claude_code.llm_request"
          - set(attributes["gen_ai.operation.name"], "chat") where name == "claude_code.llm_request"
          - set(attributes["gen_ai.tool.name"], attributes["tool_name"]) where name == "claude_code.tool"

exporters:
  otlphttp/xray:
    traces_endpoint: https://xray.${env:AWS_REGION}.amazonaws.com/v1/traces
    auth: { authenticator: sigv4auth/xray }
  otlphttp/cwlogs:
    logs_endpoint: https://logs.${env:AWS_REGION}.amazonaws.com/v1/logs
    headers:
      x-aws-log-group: ${env:AGENT_LOG_GROUP}
      x-aws-log-stream: otel-events
    auth: { authenticator: sigv4auth/logs }
  awsemf:
    namespace: aidlc/claude-code
    log_group_name: ${env:AGENT_LOG_GROUP}
  datadog:                       # Phase 1b, see the Datadog section
    api: { key: ${env:DD_API_KEY}, site: ${env:DD_SITE} }

connectors:
  datadog/connector: {}          # APM trace metrics for Datadog

service:
  extensions: [sigv4auth/xray, sigv4auth/logs]
  pipelines:
    traces:  { receivers: [otlp], processors: [resource, transform/genai, batch], exporters: [otlphttp/xray, datadog/connector, datadog] }
    logs:    { receivers: [otlp], processors: [resource, batch], exporters: [otlphttp/cwlogs, datadog] }
    metrics: { receivers: [otlp, datadog/connector], processors: [resource, batch], exporters: [awsemf, datadog] }
```

Treat the YAML as a starting point: check each component's config keys against the pinned
collector version before shipping.

### 3. AWS-side prerequisites (Terraform)

- **CloudWatch Transaction Search** enabled in the account/region (spans land in the `aws/spans`
  log group; this is what the AgentCore GenAI Observability views read).
- Execution role (`aws_iam_role.agentcore`): `xray:PutTraceSegments`, `xray:PutSpans`,
  `xray:PutSpansForIndexing`, `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`,
  `logs:DescribeLogStreams`, `cloudwatch:PutMetricData` (scoped to the namespace).
- Enable observability on the AgentCore runtime so the service-level runtime metrics/spans also
  emit; use the runtime's log group (`/aws/bedrock-agentcore/runtimes/<runtime-id>…`) as
  `AGENT_LOG_GROUP` so the agent's spans and the runtime's sit together.
- VPC mode: add VPC interface endpoints for `xray` and `logs`, or route via NAT. Datadog needs
  NAT/egress either way.
- New runtime env vars in `local.runtime_environment_variables`: the `OTEL_*` set above,
  `AGENT_LOG_GROUP`, `DEPLOY_ENV`, and (Phase 1b) `DD_SITE`; `DD_API_KEY` from Secrets Manager/SSM,
  resolved at startup like the existing MCP secrets.

### 4. Orchestrator span + `TRACEPARENT` injection (the part that makes traces useful)

Without this, every `claude -p` run is its own root trace. With it, a stage is one trace:

```text
aidlc.invocation (http-server.js, per POST /invocations)
└── aidlc.stage  (run-stage.js: intent, stage, unit, cli, model, attempt)
    ├── aidlc.git.*            (optional: init-ws, push, lane ops)
    └── claude_code.interaction   ← CLI, parented via TRACEPARENT
        ├── claude_code.llm_request
        └── claude_code.tool  (MCP aidlc.* tools, Bash, Edit, …)
```

- Add `@opentelemetry/api`, `@opentelemetry/sdk-node`, and the OTLP HTTP exporter to
  `lambda/agentcore/package.json` (or use the ADOT Node distro
  `@aws/aws-distro-opentelemetry-node-autoinstrumentation` via `--require`, which also
  instruments the AWS SDK calls to DynamoDB/S3/Lambda for free). Export to the same local collector.
- In `run-stage.js` / `resolve-conflict.js` / `one-shot.js`, start a span, then serialize it with
  `propagation.inject(context.active(), carrier)` and pass `TRACEPARENT` / `TRACESTATE` in the
  `env` passed to `runChild` / `captureChild`. `spawn.js` already merges `env` over
  `process.env`, so no change is needed there.
- Put the AgentCore session id (`X-Amzn-Bedrock-AgentCore-Runtime-Session-Id` request header) on
  the span as `session.id`, and per-call `OTEL_RESOURCE_ATTRIBUTES` on the child for
  `aidlc.intent_id`, `aidlc.stage`, `aidlc.project_id`, `enduser.id` (percent-encode values). The
  CLI's own `session.id` is its conversation id (the `--session-id` we already force), which you
  can also store in the process table to join the two.
- **Park/resume:** a resume can come hours later in a new invocation. Start a new trace for the
  resume and add a **span link** to the original stage span (persist its trace/span id in
  `V2_PROCESS_TABLE`) instead of reusing the old `TRACEPARENT`, which would create a trace hours long.
- **Flush:** the CLI flushes on clean exit with a short bound. The 1 s export intervals plus a
  collector that outlives the child cover the normal case. Call `sdk.shutdown()` / `forceFlush()` on
  the Node side when the invocation ends, since the microVM can be reaped after 900 s idle.

### 5. Version bump

`2.1.246` supports traces, `TRACEPARENT`, and `CLAUDE_CODE_OTEL_DIAG_STDERR`. Newer releases add
things we want, so bump to **≥ 2.1.283**:

| Feature | Min version |
| - | - |
| `parent.source` on `claude_code.interaction` (confirms TRACEPARENT took) | 2.1.268 |
| `query_source_safe`, `error_class`, `tool_name_safe`, `bash_command_class` | 2.1.268 |
| Bedrock `request_id` from `x-amzn-requestid` (join spans to Bedrock logs) | 2.1.282 |
| `tool.output` span event for **MCP** tool results (our `aidlc` MCP server) | 2.1.283 |

### 6. Content gating (decide explicitly)

Defaults are structural only. `OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`,
`OTEL_LOG_TOOL_CONTENT`, `OTEL_LOG_RAW_API_BODIES` put prompts, file paths, commands, and file
contents into CloudWatch/Datadog. Customer code and requirements flow through these stages, so
make it a per-environment Terraform variable defaulting to off (maybe `TOOL_DETAILS` on in dev).

### 7. Other CLIs (Kiro, OpenCode, Codex)

The collector and the orchestrator span cover them immediately: every stage gets an
`aidlc.stage` span with duration, exit code, model, and token usage from the parsers
(`codex-parser.js`, `opencode-parser.js`, `parseKiroCredits`). Native per-LLM-call spans depend
on each CLI's own OTel support; check each separately and point any that support OTLP at
`127.0.0.1:4318`.

### Phase 1 deliverables checklist

- [ ] `otelcol-contrib` in Dockerfile + `otel/collector.yaml` + startup supervision
- [ ] OTel env in `claudeDriver.envForAuth` (or a new `envForTelemetry`), gated by `AIDLC_OTEL_ENABLED`
- [ ] Node OTel SDK in `http-server.js`; spans in run-stage / resolve-conflict / one-shot; TRACEPARENT injection
- [ ] Terraform: IAM, env vars, Transaction Search, VPC endpoints, DD secret
- [ ] Claude Code bump to ≥ 2.1.283
- [ ] Tests: `drivers.test.js` asserts telemetry env; `run-stage.test.js` asserts TRACEPARENT is passed to `runChild`
- [ ] Verify in CloudWatch: one trace per stage, `claude_code.interaction.parent.source == env`

## Phase 2 — optional `claude-sdk` driver (Agent SDK alongside `claude -p`)

Not needed for telemetry. Worth it for:

- **In-process hooks** (`PreToolUse`, `PostToolUse`, `SubagentStart`, …) as JS callbacks: emit
  custom spans/events, enforce policy, or redact without shelling out to hook scripts.
- **`canUseTool` callback** instead of `--permission-mode bypassPermissions`.
- **Typed message stream** (`SDKMessage`) instead of parsing `stream-json` stdout; the final
  `result` message carries `usage`, `total_cost_usd`, `num_turns`, `session_id`.
- **In-process SDK MCP servers** (`createSdkMcpServer`) could replace the stdio `aidlc` MCP
  subprocess (`V2_MCP_ENTRY`) and share the orchestrator's clients, so MCP tool calls get
  in-process spans too.
- `AbortController` cancellation, `interrupt()`, streaming input, `stderr` callback for OTel diagnostics.
- Automatic `TRACEPARENT` injection from the active span.

### Design

The driver contract today is "return `{ command, args, env }` for `spawn`". The SDK is a library
call, so add an execution kind instead of forcing it through `runChild`:

1. `drivers.js`: add `claudeSdkDriver` with `kind: 'sdk'` (existing drivers implicitly `'spawn'`),
   `contextKey: 'mcpConfigPath'` (reuse the materializer), and
   `buildQueryOptions({ prompt, mcpConfigPath, model, allowedTools, sessionId })` returning SDK options.
2. New `cli/sdk-runner.js`: `runSdk({ prompt, options, onMessage })` iterates `query()` and resolves
   the **same contract** as `runChild`: `{ exitCode, stderrTail }`, mapping `result.subtype`
   (`success` / `error_*`) to an exit code, so `run-stage.js` stays nearly unchanged.
3. `run-stage.js` / `resolve-conflict.js` / `one-shot.js`: branch on `driver.kind`.
4. `selectCli` / `SUPPORTED_CLIS` / `discover.js`: add `claude-sdk`, available when the package
   resolves; select it from the same admin/project setting that picks the CLI today.

Flag → option mapping (verify names against the pinned SDK version's type definitions):

| `claude -p` today | Agent SDK (TypeScript) |
| - | - |
| prompt on stdin | `prompt` (string or async iterable) |
| `--mcp-config <file>` | `mcpServers` (load the materialized JSON's `mcpServers` object) |
| `--permission-mode bypassPermissions` | `permissionMode: 'bypassPermissions'` (recent versions also require `allowDangerouslySkipPermissions: true`), or `canUseTool` |
| `--model` | `model` |
| `--allowedTools a,b` | `allowedTools: ['a','b']` |
| `--session-id <uuid>` (new session) | check the pinned SDK for a session-id option; otherwise read `session_id` from the `system/init` message |
| `--resume <uuid>` | `resume: '<uuid>'` |
| `--output-format stream-json --verbose` | native message iteration |
| driver `env` (Bedrock, `CLAUDE_CONFIG_DIR`, OTel) | `env`. **In TS, `env` replaces the inherited env**: pass `{ ...process.env, ...driverEnv }` |
| `cwd` | `cwd` |

Packaging: add `@anthropic-ai/claude-agent-sdk` to `lambda/agentcore/package.json`. The SDK ships
its own CLI build; either accept that (two Claude Code versions in the image) or set
`pathToClaudeCodeExecutable` to the globally installed, pinned `claude` so one version is used.
Park/resume durability is unchanged: the SDK runs the same CLI, so the JSONL conversation store
under `CLAUDE_CONFIG_DIR=/mnt/workspace/.claude` still works.

Rough size: driver + runner + branching ≈ 300–500 LOC plus tests. Ship behind the selector so
`claude` (CLI) and `claude-sdk` run side by side, compare traces/outcomes, then decide on a default.

## Datadog

**Same telemetry, not the same problem.** The CLI emits standard OTLP; Datadog ingests OTLP. The
issue on AWS was auth (SigV4), and the in-container collector solves it for both backends:

| Option | Notes |
| - | - |
| **A. Collector fan-out (recommended)** | Add the `datadog` exporter + `datadog/connector` to the Phase 1 collector. One egress point, one config, no Datadog code in the app. API key from Secrets Manager. |
| B. Datadog Agent in the container | Accepts OTLP on 4317/4318 too, but it's heavy for a per-session microVM and duplicates the collector. |
| C. Agentless OTLP intake straight to Datadog | Check current per-signal support for your Datadog site. Loses the AWS fan-out; you'd still need the collector for CloudWatch. |
| ✗ `dd-trace` into the Claude child | Don't. The CLI is a bundled process; use its OTLP output. `dd-trace` in `http-server.js` is possible, but OTel API + OTLP keeps one instrumentation for both backends. |

Datadog specifics:

- **Unified service tagging**: set `service.name`, `deployment.environment`, `service.version` as
  resource attributes; Datadog maps them to `service`, `env`, `version`.
- **LLM Observability**: Datadog maps OTel GenAI semantic-convention spans (`gen_ai.*`) into LLM
  Observability. Claude Code already sets `gen_ai.system`, `gen_ai.request.model`,
  `gen_ai.response.id`, `gen_ai.response.finish_reasons`, `gen_ai.tool.call.id`; the
  `transform/genai` processor above fills in token usage. Confirm which attributes your Datadog
  plan requires.
- **Trace ↔ log correlation**: with `TRACEPARENT` set, CLI log events carry `trace_id`/`span_id`
  (Phase 1 step 4), so Datadog links logs to traces automatically.
- **Cost of both**: everything is sent twice. Use the collector's `filter` /
  `probabilistic_sampler` per pipeline if Datadog volume or cost matters (e.g. all traces to X-Ray,
  sampled to Datadog; metrics to both).

## Open questions to verify during Phase 1

1. Confirm `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` does not suppress OTLP export to the local
   collector (step 1 proves it either way; if it does, replace it with the narrower
   `DISABLE_AUTOUPDATER` / `DISABLE_ERROR_REPORTING` / `DISABLE_TELEMETRY` set).
2. Which views of the **AgentCore GenAI Observability** dashboard light up from Claude Code spans
   alone (they're in `aws/spans` either way; the curated agent/session views key on resource
   attributes and `session.id`). Adjust the `resource` / `transform` processors to fit.
3. Collector memory/CPU overhead in the microVM; tune `batch` and `memory_limiter`.
4. Whether to keep content gates off in prod (security review).
