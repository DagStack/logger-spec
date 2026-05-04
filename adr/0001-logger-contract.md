# ADR-0001: Logger contract — OTel-compatible structured logging

- **Status:** accepted
- **Revision:** 1.2 (2026-05-03)
- **Date:** 2026-04-19
- **Architect review:** ai-systems-architect — initial round 2026-04-19; must-fix round 2026-04-19 applied (timestamp naming aligned with OTel spec, trace_id/span_id wire encoding, severity_text canonical set, exception contract stub, observed_time ownership, `dagstack.logger.internal` isolation, reserved-domains tier split); v1.1 amendment 2026-05-03 — §10.4 added (public Phase 1 redaction-config API for M3); §10.1 cross-reference updated; v1.2 amendment 2026-05-03 — §3.4 expanded with explicit auto-injection contract + per-language defaults (M2)
- **Supersedes:** —
- **Related:**
  - `dagstack/config-spec` ADR-0001 — the logger reads its own configuration through the config-spec API.
  - `dagstack/plugin-system-spec` ADR-0001 — plugin architecture core (the logger's instrumentation_scope is associated with a plugin manifest); ADR-0003 §6 Progress / Checkpoint sinks — §5.3 Progress events of this spec absorb the Progress sink as a particular case of a convention over LogRecord.
  - OpenTelemetry Log Data Model (v1.24+), OTel Context API (W3C Trace Context).
  - **OpenTelemetry GenAI semantic conventions** (https://opentelemetry.io/docs/specs/semconv/gen-ai/) — attribute namespaces for LLM/agent/MCP observability; the dagstack AI-agent conventions (§5.5) are a conformant superset.

## Context

Today, dagstack applications (a pilot consumer in Python and future ones in TS/Go) rely on ad-hoc logging: Python `logging` + `structlog`, TS `pino`/`winston`, Go `zap`/`slog`. The result:

- **Non-portable log records** — schemas and wire formats differ, and the central observability platform (Loki / CloudWatch / Elastic) receives inconsistent streams.
- **No context propagation** — `trace_id`/`span_id` are injected by hand, `tenant_id` is lost across service boundaries, and correlation between traces and logs is best-effort.
- **OTel fragmentation** — every application configures its own OTLP exporter separately, with versioning and batch tuning out of sync.
- **Scattered redaction** — every team writes its own redactor for API keys / PII and often forgets to cover newly added fields.
- **No runtime reconfiguration** — changing the log level requires a restart, even when config-spec supports hot reload.

**Goal of this ADR** — fix a cross-language logger contract: a wire format based on the OTel Log Data Model, an API surface, a sink-adapter roadmap, and integration with `config-spec` for configuration and runtime reconfiguration. Per-language bindings (`dagstack/logger-python`, `-typescript`, `-go`) implement the contract idiomatically.

### Prior art

- **OpenTelemetry Logs** (spec + SDK) — Log Data Model (severity, body, attributes, resource, instrumentation_scope, trace_id/span_id/trace_flags, timestamp_unix_nano, observed_timestamp_unix_nano). Adopted as the wire-format baseline.
- **Python `logging` + `structlog`** — named loggers + dot-hierarchy + processor chain + structured context. We borrow the hierarchy and processor model.
- **Go `slog`** (standard library, stable since 1.21) — structured logging with `Record` + `Handler` interface. Validates the design on a statically typed language.
- **pino** (Node.js) — async-first, low-overhead, child loggers with bindings. Validates the async-oriented design.
- **rs/zerolog** (Go) — zero-allocation structured logging. Benchmark reference.
- **Dagster event log** — the idea that "a log is not a stream but a tree of structured events around running operations." From Dagster we borrow: (a) operation-level semantic conventions (§5.1), (b) typed events with a schema registry (§5.2), (c) progress-as-event convention (§5.3), (d) metadata type hints for rich UI rendering (§5.4), (e) scoped logger override per run (§6). **What we do not borrow**: pipelines/jobs/steps as first-class concepts (orchestrator scope, not the logger's), the storage layer (a backend concern), or assets (data-domain-specific). The wire format remains the OTel LogRecord — Dagster-style richness is implemented as a convention over `attributes`, not as a new structure.
- **OTel GenAI semantic conventions** (plus MCP conventions, see https://opentelemetry.io/docs/specs/semconv/gen-ai/) — a community-driven set of attribute keys for LLM-call observability (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.operation.name`) and the Model Context Protocol (`mcp.server.name`, `mcp.method.name`, `mcp.tool.name`, `mcp.request.id`). The dagstack AI-agent conventions (§5.5) are based on this standard — we **commit to conformance** and only extend with our own namespaces (`rag.*`, `agent.*`, `prompt.*`) where OTel GenAI has not yet covered the need.
- **OpenLLMetry** (Traceloop), **Langfuse**, **LangSmith**, **Helicone** — production-deployed AI observability tools, all converging on the `gen_ai.*` namespace plus per-step spans. We follow that convergence rather than inventing a parallel standard.

## Decision

### 1. Log Data Model (wire format)

LogRecord is structurally identical to the **OTel Log Data Model v1.24**. Field names in the internal model match the OTel normative spec (proto field names):

```
LogRecord {
  time_unix_nano:          uint64                  # emit time, nanoseconds since Unix epoch (OTel: TimeUnixNano)
  observed_time_unix_nano: uint64                  # ingest time at the sink (OTel: ObservedTimeUnixNano)
  severity_number:         int (1..24)             # OTel severity numbering
  severity_text:           enum string             # one of 6 canonical values (§2); MUST match the OTel-recommended set
  body:                    Value                   # primary message; string or structured (map/array/scalar)
  attributes:              Map<string, Value>      # per-record key-value context
  resource:                Resource                # process-/host-/service-level attributes (service.name, etc.)
  instrumentation_scope:   InstrumentationScope    # logger name + version (§4.1)
  trace_id:                bytes(16) | null        # W3C Trace Context, 16 random bytes (OTel TraceId)
  span_id:                 bytes(8)  | null        # W3C Trace Context, 8 random bytes (OTel SpanId)
  trace_flags:             uint8                   # W3C flags (sampled bit, etc.)
}
```

`Value` is a recursive sum type: `string | int | float | bool | null | Map<string, Value> | Value[]`. The same shape backs `ConfigTree` in `config-spec` §8.1 (`ConfigSource.load() → ConfigTree`); spec consumers MAY reuse a single language-level type to represent both.

**Wire-format outputs** (sink-specific — encoding differs across wire formats while the internal model stays the same):

- **OTLP protobuf** (Phase 2 `OTLPSink` gRPC/HTTP): `time_unix_nano`, `observed_time_unix_nano` are `fixed64`; `trace_id` is `bytes` (16); `span_id` is `bytes` (8). Native OTel wire.
- **OTLP JSON** (`OTLPSink` HTTP+JSON protocol, `FileSink` OTLP mode, `ConsoleSink` JSON mode): per **OTel JSON encoding** — `timeUnixNano` / `observedTimeUnixNano` as **string** decimal (per the protobuf JSON spec for 64-bit ints); `traceId` / `spanId` as **lowercase hex strings** (32 / 16 hex chars respectively, zero-padded). Keys are camelCase per OTel JSON conventions.
- **dagstack JSON-lines** (`FileSink` default mode, `ConsoleSink` wire mode): for dagstack-specific payloads, **snake_case** field names (`time_unix_nano`, `observed_time_unix_nano`, `trace_id`, `span_id`) — a style aligned with config-spec Canonical JSON. Timestamps are integer nanoseconds (Canonical JSON allows int64 as a native number without string-wrapping; consumers MUST handle bigints when reading the wire format). `trace_id`/`span_id` are lowercase hex strings (32/16 chars). Canonical JSON rules (sorted keys recursively, UTF-8, LF, no trailing newline) apply.
- **Human-pretty** (`ConsoleSink` dev mode): colorized single-line or multi-line — **presentation only**, not a wire format; a parser is not required to reconstruct LogRecord from the pretty output.

**A binding MUST** document which wire format is the default for each sink; switching between OTLP JSON and dagstack JSON-lines goes through sink config. Conformance fixtures (the §12 of config-spec pattern) cover both.

**`observed_time_unix_nano` ownership:** the producer (the binding's `emit()`) leaves `observed_time_unix_nano = null`. The sink **MUST** set the current time at ingestion when the field is null, prior to wire serialization. This guarantees that FileSink / OTLPSink wire output always carries an observed timestamp; in an in-memory pipeline (InMemorySink) both timestamps may coincide.

### 2. Severity levels

OTel severity numbering (1-24) is used for `severity_number`. For `severity_text` **exactly one of the 6 canonical OTel-recommended strings MUST be used** — `"TRACE"`, `"DEBUG"`, `"INFO"`, `"WARN"`, `"ERROR"`, `"FATAL"`. This is required so that backends (Grafana, Honeycomb, Datadog) can correctly filter by severity using exact-match on `severity_text`. Numeric granularity is carried in `severity_number`.

| `severity_number` range | `severity_text` | Semantic |
|---|---|---|
| 1-4 | `"TRACE"` | Fine-grained diagnostic |
| 5-8 | `"DEBUG"` | Debug-level diagnostic |
| 9-12 | `"INFO"` (default INFO=9) | Informational |
| 13-16 | `"WARN"` | Warning conditions |
| 17-20 | `"ERROR"` | Error requiring attention |
| 21-24 | `"FATAL"` | Critical — application-level failure |

A binding exposes the primary names (`trace`, `debug`, `info`, `warn`, `error`, `fatal`) as logger methods with `severity_number` = 1, 5, 9, 13, 17, 21 respectively; intermediate values (2-4, 6-8, 10-12, ...) go through the generic `log(severity_number, ...)` and keep the same `severity_text` bucket.

Mapping to syslog (0-7) / Python `logging` (10/20/30/40/50) / Go `slog` (INFO/WARN/ERROR numeric levels) lives in `_meta/severity_mapping.yaml` (the source of truth for emitters).

### 3. Logger API (language-agnostic contract)

The contract is abstract pseudo-code. Bindings implement it idiomatically (see §3.5).

#### 3.1. Named loggers and hierarchy

```
logger := Logger.get(name: string, version?: string): Logger
```

`name` is a dot-notation path (`dagstack.plugin_system`, `dagstack.plugin_system.config`). Hierarchy is parent/child via dot-prefix; the level and attached sinks are inherited from the parent unless explicitly overridden.

`(name, version)` is the LogRecord's instrumentation_scope. For plugins, the recommendation is `name = plugin.id` from the PluginManifest, `version = plugin.version`.

**Root logger** — `name = ""` (empty string) or `"root"` (binding-specific). Default level is `INFO`; the default sink is `ConsoleSink` with the dev-mode pretty printer (the binding decides when to enable dev vs JSON: typically by TTY presence).

#### 3.2. Primary log methods

```
logger.trace(body, attrs?)
logger.debug(body, attrs?)
logger.info(body, attrs?)
logger.warn(body, attrs?)
logger.error(body, attrs?)
logger.fatal(body, attrs?)
logger.log(severity_number, body, attrs?)        # generic, for TRACE2/INFO3/etc.
logger.exception(err, attrs?)                    # ERROR severity + auto stack-trace capture
```

- `body` is a string (a human message) or `Map<string, Value>` (a structured body). A string is the most common case.
- `attrs` is `Map<string, Value>` carrying per-record context. Always structured, never positional.

**Exception contract (OTel semconv-compatible).** `logger.exception(err, attrs?)` MUST populate attributes per the OTel `exception.*` semantic conventions:

- `exception.type` — fully qualified type name (`"ValueError"` in Python, `"TypeError"` in TS, `"*net.OpError"` in Go, `"com.example.FooException"` in Java).
- `exception.message` — `str(err)` / `err.message` / `err.Error()`.
- `exception.stacktrace` — opaque string; formatting is binding-specific (Python `traceback.format_exception`, TS `err.stack`, Go `runtime.Callers` + symbolication, Java `Throwable.printStackTrace()`). Wire representation is a UTF-8 string; backends parse it with their own heuristics.
- `exception.escaped` — boolean, optional: `true` when an exception crossed a span boundary uncaught.

A binding MAY add non-OTel debug attributes (chained causes, Python `__cause__`/`__context__`) under its own `dagstack.exception.*` namespace. The full convention for stacktrace format is open question #3 and will most likely be fixed in v1.1 as a separate `_meta/conventions/exception.yaml`.

**Methods are non-blocking.** `logger.info(...)` returns immediately (§13).

#### 3.3. Contextual bindings (child loggers)

```
scoped := logger.with(attrs: Map<string, Value>): Logger
```

Returns a child logger with pre-attached `attrs` — every record from the child merges those attrs into its `attributes`. Pino-style bindings / structlog `bind`. Children are cheap (a reference to the parent plus a delta map) and safe for concurrent use.

**Child-logger inheritance:**

- Level: inherited; override via `scoped.setLevel(severity_number)`.
- Sinks: inherited (writes to the same sinks as the parent).
- Name: same as the parent (the dot-prefix does not change through `with`).
- Attributes: merged at emit time.

#### 3.4. Context propagation (mandatory)

Loggers **MUST** inject the following fields into the LogRecord whenever the binding's mode (§3.4.1) and the `auto_inject_trace_context` flag (§3.4.2) make the active context observable to the emit call:

- `trace_id`, `span_id`, `trace_flags` — from the active **OTel Context** in thread/async-local state.
- `baggage.tenant_id` (if present) → `attributes["tenant.id"]`.
- Other W3C Baggage entries listed in `_meta/baggage_keys.yaml` (`tenant.id`, `request.id`, `user.id` by default) are injected into `attributes` under the same key.

**A binding MUST** use the OTel Context API (Python `opentelemetry.context`, TS `@opentelemetry/api` `context`, Go `context.Context`) as the source of active trace info.

##### 3.4.1. How the active context reaches the emit

Two modes are recognized:

- **Auto-inject mode** — the emit method takes no explicit `ctx`/`Context` argument and the binding reads the active context from the OTel Context API (Python `opentelemetry.context.get_current()` — which delegates to `contextvars`; TypeScript `@opentelemetry/api`'s global `context.active()` driven by `AsyncLocalStorage` or equivalent).
- **Explicit-ctx mode** — the emit method takes an explicit context argument as its first parameter (Go `logger.InfoCtx(ctx, ...)`). The binding extracts trace state from the supplied context only; ambient state is not consulted.

**A binding MUST support at least one of these modes for every severity method**, and **MAY** support both. Per-language defaults (§3.5) document which mode is canonical per binding.

A binding that claims to support the auto-inject mode **MUST** implement ambient-context lookup that respects an active OTel context — not a stub that silently produces records without trace IDs. A binding whose runtime has no idiomatic ambient-context primitive (e.g. Go) **MUST** declare auto-inject as **unsupported** in its per-language ADR (§3.5) rather than implementing it as a no-op.

##### 3.4.2. Cross-binding parity flag — `auto_inject_trace_context`

The mode is a property of the binding's API surface (§3.4.1); the flag below controls behaviour **within** the auto-inject mode and is a no-op for emits that go through the explicit-ctx path.

The `configure(...)` surface (§9) **MUST** accept an `auto_inject_trace_context: bool` option (per-language naming follows §3.5 idioms — `auto_inject_trace_context` in Python YAML / `autoInjectTraceContext` in TS / `WithAutoInjectTraceContext(bool)` in Go). The flag affects only the auto-inject mode:

- When `true`, severity methods that **do not** take an explicit context (`logger.info(...)` in Python / TS, not the explicit-ctx variant) **MUST** read the ambient context and inject `trace_id`/`span_id`/`trace_flags` if a sampled span is active.
- When `false`, the ambient lookup is skipped — non-`*Ctx` emits produce records without trace IDs (the `*Ctx` variant, if present, still injects from the supplied `ctx`).
- The default per binding is documented in the per-language ADR (§3.5):
  - **Python**: `true` (idiomatic — contextvars are the standard ambient channel).
  - **TypeScript**: `true` (idiomatic — `@opentelemetry/api` exposes a global active-context API and Node ≥16 has `AsyncLocalStorage`).
  - **Go**: `false` (idiomatic — Go avoids implicit context; explicit `*Ctx` methods stay primary). The Go binding **MAY** declare auto-inject as unsupported per §3.4.1; in that case `WithAutoInjectTraceContext(true)` MUST surface a configuration error or warning rather than silently no-op.

This flag is the cross-binding parity lever: deployments that need byte-identical wire output across Python/TypeScript/Go MUST set the flag explicitly to the same value in all three bindings.

##### 3.4.2.1. OTel Logs Bridge alignment

This flag does not conflict with OTel's Logs Bridge or LogRecordExporter contracts: those specify wire format and exporter behaviour, not how a logger binding obtains trace state. When `auto_inject_trace_context = true`, the resulting LogRecord matches the OTel SDK's canonical default exactly; when `false`, the record is still OTel-compatible at the wire level (`trace_id`/`span_id` are simply absent, which the OTel Log Data Model allows). Bindings that wrap the OTel Logs SDK directly (e.g. `LoggingProvider.bridge()`) **MAY** treat this flag as a binding-level pre-filter applied before the OTel SDK call.

##### 3.4.3. Opt-out for an individual emit

For fire-and-forget emits not bound to caller context, the per-mode opt-out idioms are:

| Mode | Opt-out idiom |
|---|---|
| auto-inject | `logger.without_context().info(...)` — detached child handle skipping ambient lookup |
| explicit-ctx | call the non-`*Ctx` variant, or pass `nil` / `context.Background()` to `*Ctx` |

The two forms **MUST** be semantically equivalent: a record emitted via either path within an active span produces a wire record with `trace_id`/`span_id`/`trace_flags` absent. In Go, `nil` and `context.Background()` are accepted interchangeably for this purpose — both produce the same wire output (fields absent), so conformance fixtures that exercise the explicit-ctx opt-out path may use either.

The auto-inject opt-out (`without_context()`) returns a detached child handle. Not inherited further (children default to re-enabled).

##### 3.4.4. Conformance fixture

`_meta/fixtures/trace_context_propagation.json` exercises both modes:

- A sequence with an active span → expects `trace_id` / `span_id` / `trace_flags` populated.
- The same sequence with `auto_inject_trace_context = false` (auto-inject bindings) or no-ctx (explicit-ctx bindings) → expects the trace fields absent.
- The opt-out path (`without_context()` / no-ctx) inside an active span → expects trace fields absent.
- No active span (auto-inject mode) → trace fields absent regardless of `auto_inject_trace_context`.

Cross-binding identity is asserted by comparing the canonical-JSON wire bytes of the redaction-stripped LogRecord against the fixture's `expected` block.

#### 3.5. Per-language idioms (bindings)

- **Python:** sync and async variants (`logger.info(...)`, `await logger.ainfo(...)`), dict-based attrs, `logger.exception()` accepts the active exception from `sys.exc_info()`.
- **TypeScript:** `log.info(body, {attrs})`, async-friendly (Promises returning immediately). Child bindings via `log.child({...})` — a conventional alias for `with`.
- **Go:** `slog`-style `logger.Info(msg, slog.String("k", "v"), ...)` variadic attrs; `context.Context` as the first argument: `logger.InfoCtx(ctx, ...)`.
- **Rust:** `tracing`-style structured records via macros, but the spec-level API is the same.

A binding fixes its choice in a per-language ADR. Core semantics (fire-and-forget, context propagation, hierarchy, record shape) are identical.

#### 3.6. Error handling

- Emit errors (malformed attrs, a record dropped due to buffer overflow) **do not raise exceptions** to the caller. The logger self-logs errors on its binding's diagnostic channel (Python `logging.warning` on `dagstack.logger.internal`, TS `console.warn`, Go a structured log on the `log/slog` default handler).
- Shutdown errors (flush timeout, sink close failure) are returned through `logger.flush(timeout) -> FlushResult { success, partial, failed_sinks[] }` and `logger.close()`.

### 4. Resource and instrumentation scope

#### 4.1. InstrumentationScope

Each logger is associated with an instrumentation_scope (`name`, `version`, `attributes?`). Sources:

- `Logger.get("dagstack.rag.retriever", "1.4.2")` — explicit.
- Plugin context: when a binding is integrated with `plugin-system-spec`, the logger receives its scope from `PluginManifest { id, version }` (`logger = pluginContext.logger`).

#### 4.2. Resource

Process-level attributes shared across all loggers in the same process (OTel Resource):

- `service.name`, `service.version`, `service.instance.id`, `deployment.environment`, `host.name`, `process.pid`, `telemetry.sdk.{name,version,language}`.

A binding detects Resource through:

1. OTel autodetect (`OTEL_RESOURCE_ATTRIBUTES` env var, OTel Resource Detectors).
2. Explicit config from the `logging.resource.*` section (see §9 config-spec integration).
3. Merge: env-detected ∪ config-supplied (config wins).

### 5. Semantic conventions

The core wire format (§1) covers the transport layer. On top of it the spec publishes a **set of semantic conventions** — standardized keys in `attributes` and required fields for certain event types. Bindings validate conventions at emit time, and backends receive a consistent data model across all dagstack applications.

All conventions live as the source of truth in `_meta/conventions/` (see §5.6) and are emitted into per-language constants via `emitters/`.

#### 5.1. Operations

Any business-logic operation that runs longer than a single tick (index_repo, query_rag, load_plugin, describe_repo, embed_batch) carries the following attributes in its LogRecords and enclosing span:

| Attribute | Type | Semantic |
|---|---|---|
| `operation.name` | string | A stable, human-readable identifier (snake_case): `index_repo`, `query_rag`, `load_plugin`, `describe_repo`. Does not change between deployments. The source of truth for UI lookup. |
| `operation.id` | string | A unique UUID for the operation instance. De-facto an alias of span_id, but with a stable name (the UI looks up by `operation.id` without knowing about tracing). |
| `operation.kind` | enum string | `indexing`, `query`, `lifecycle`, `hook`, `maintenance`, `admin`. Extended through `_meta/conventions/operation_kinds.yaml`. |
| `operation.parent.id` | string? | The parent operation (when there is a hierarchy, e.g. `query_rag` → `semantic_search` → `embed_batch`). |
| `operation.status` | enum? | On a `completed` event: `ok`, `error`, `cancelled`, `timeout`. |
| `operation.duration_ms` | int? | On a `completed` event: milliseconds from start to end. |

**Required:** `operation.name` and `operation.id` are mandatory in every LogRecord emitted within an operation boundary. A binding MUST provide a helper `logger.operation(name, kind)` to set the active operation in context and auto-inject these attributes into all child LogRecords and spans.

#### 5.2. Typed events

Instead of an unstructured body with an arbitrary message, the convention is to **emit typed events** with required `event.domain` and `event.name`:

```
attributes: {
  event.domain: "rag",
  event.name:   "chunk_retrieved",
  event.schema_version: "1.0",
  rag.chunk.id: "abc123",
  rag.chunk.score: 0.87,
  rag.chunk.repo: "order-service",
  ...
}
```

Schemas live in `_meta/events/<domain>.yaml`:

```yaml
# _meta/events/rag.yaml
events:
  chunk_retrieved:
    schema_version: "1.0"
    severity_hint: DEBUG
    required_attributes:
      - rag.chunk.id
      - rag.chunk.score
      - rag.chunk.repo
    optional_attributes:
      - rag.chunk.path
      - rag.chunk.line_range
    description: "A single chunk passed the min_score threshold in semantic_search."
```

**A binding MUST:**

1. On `logger.emitEvent(domain, name, attrs)`, validate the presence of required_attributes.
2. Inject `event.domain`, `event.name`, `event.schema_version` automatically.
3. Use the default severity from `severity_hint`, overridable via an explicit `severity` arg.
4. Provide conformance tests (the §9.1.1 of config-spec pattern) — golden event fixtures in `conformance/events/`.

**Reserved domains** are split into two tiers:

- **Core-reserved** (always enforced; bindings MUST reject emits from user code on these domains): `dagstack.*`, `operation.*` (§5.1), `progress.*` (§5.3). These are the mandatory domains of every dagstack application; their semantics are fixed in the core spec.
- **Extension-pack-reserved** (enforced **only when the extension pack is loaded**): the AI-agent extension (§5.5) activates the reservation for `gen_ai.*`, `ai_agent.*`, `rag.*`, `agent.*`, `prompt.*`, `mcp.*`. In a non-AI application these namespaces are simply user-defined, with no warnings. Future extension packs (e.g., data-pipeline observability) will add their own reserved namespaces under the same model.

**User-defined domains** — any domain without a core-reserved prefix that does not collide with active extension-pack reservations. The convention is reverse-DNS style (`com.example.myapp.checkout`) to avoid collisions between independent applications.

#### 5.3. Progress events

Progress reporting (indexing 10000 files, an embedding batch, a training epoch) is a convention over LogRecord, **not** a separate sink system. It absorbs the Progress sink from `dagstack/plugin-system-spec`.

```
event.domain:  "progress"
event.name:    "tick"  |  "started"  |  "completed"  |  "failed"
severity:      DEBUG (tick) | INFO (started/completed) | ERROR (failed)

attributes:
  progress.current:   int           # completed units
  progress.total:     int?          # total units if known; null for unknown-total streams
  progress.unit:      string        # "files", "chunks", "tokens", "bytes", "items"
  progress.phase:     string?       # "discovery", "chunking", "embedding", "upserting", "describing"
  progress.rate:      float?        # unit/sec (sliding window, binding-computed)
  progress.eta_ms:    int?          # estimated ms to completion (when total is known)
  operation.id:       string        # required — progress is always bound to an operation (§5.1)
```

**Rate-limiting:** at high tick frequency a binding MUST provide a helper with rate-limiting (e.g., `logger.progress(current, total, unit, min_interval_ms=500)` to drop ticks more frequent than `min_interval_ms`). `started` / `completed` / `failed` are always emitted without dropping.

**UI rendering:** by convention, backends (Grafana / Loki UI / a custom dashboard) treat progress as a stream of ticks with the same `operation.id` and render a progress bar.

#### 5.4. Metadata type hints

Attributes whose values should render richly in an observability UI carry a **suffix hint**:

| Suffix | Value type | UI rendering |
|---|---|---|
| `*.url` | string (http(s) URL) | clickable link |
| `*.path` | string (fs path) | monospace, copyable |
| `*.markdown` | string (Markdown) | rendered Markdown block |
| `*.table_json` | string (canonical JSON array of objects) | tabular view |
| `*.code` | string (source code, language in `*.code.lang`) | syntax-highlighted block |
| `*.duration_ms` | int | duration formatted (e.g., "12.3s") |
| `*.bytes` | int | human-readable size (e.g., "1.2 MB") |
| `*.timestamp` | int (unix_nano) or string (ISO-8601) | datetime with timezone |
| `*.image.url` | string | embedded image preview |

Examples:

```
gen_ai.response.content.markdown: "# Answer\n\nThe function foo() ..."
rag.chunk.path: "src/rag/retriever.py"
rag.chunk.source.url: "https://git.example.com/.../retriever.py#L42"
llm.prompt.tokens.bytes: 16384
```

Suffixes are a **convention, not a strict schema**. A backend without support simply renders the value as a plain string. The full list of suffixes lives in `_meta/conventions/metadata_hints.yaml`.

#### 5.5. AI-agent conventions (optional extension pack)

The core logger-spec is domain-agnostic. This section is an **optional convention pack** for AI/LLM applications (a pilot consumer, agentic pipelines). Applications without AI pipelines ignore §5.5 entirely.

##### 5.5.1. Conformance to OpenTelemetry GenAI

The dagstack AI-agent conventions **commit to conformance with OTel GenAI semantic conventions** (https://opentelemetry.io/docs/specs/semconv/gen-ai/). We:

- **Use OTel attribute names 1:1** for OTel-covered namespaces (`gen_ai.*`, `mcp.*`). We do not introduce parallel keys for the same concept.
- **Extend with our own namespaces** only where OTel GenAI has not yet stabilized a convention (`rag.*`, `agent.*`, `prompt.*`). When an OTel equivalent appears, we migrate to the OTel convention in the next major version of logger-spec, with a deprecation period for the old keys.
- **Follow OTel MCP conventions** for Model Context Protocol observability (`mcp.server.name`, `mcp.method.name`, `mcp.tool.name`, `mcp.request.id`, `mcp.response.is_error`, `mcp.transport`).
- `_meta/conventions/genai_otel_mapping.yaml` is the source-of-truth mapping between our conventions and the current OTel GenAI spec. It is updated on every OTel GenAI release.

##### 5.5.2. Attribute namespaces

**`gen_ai.*`** (OTel GenAI — used as is):

| Attribute | Semantic |
|---|---|
| `gen_ai.system` | `openai`, `anthropic`, `gemini`, `vllm`, `openrouter`, `bedrock`, ... |
| `gen_ai.operation.name` | `chat`, `text_completion`, `embeddings`, `tool_call` |
| `gen_ai.request.model` | model id as requested |
| `gen_ai.response.model` | model id as returned (may differ — e.g., OpenAI versioning) |
| `gen_ai.request.temperature` / `.top_p` / `.max_tokens` / `.frequency_penalty` / `.presence_penalty` | sampling params |
| `gen_ai.usage.input_tokens` / `.output_tokens` / `.cache.read_input_tokens` | token accounting |
| `gen_ai.response.finish_reasons[]` | `stop`, `length`, `tool_calls`, `content_filter`, `function_call` |
| `gen_ai.response.id` | provider-assigned response id |
| `gen_ai.tool.name` / `gen_ai.tool.call.id` | tool dispatch |
| `gen_ai.prompt.<N>.role` / `.content` | conversational messages (subject to truncation, see §5.5.5) |

**`mcp.*`** (OTel MCP — used as is):

| Attribute | Semantic |
|---|---|
| `mcp.server.name` | MCP server identity |
| `mcp.server.version` | |
| `mcp.method.name` | `tools/call`, `tools/list`, `resources/list`, `resources/read`, `prompts/get`, `logging/setLevel`, ... |
| `mcp.request.id` | JSON-RPC request id |
| `mcp.tool.name` | tool name when `method.name = "tools/call"` |
| `mcp.response.is_error` | boolean |
| `mcp.transport` | `stdio`, `http+sse`, `streamable_http` |

**`rag.*`** (dagstack-specific, not covered by OTel GenAI):

| Attribute | Semantic |
|---|---|
| `rag.query.original` / `rag.query.rewritten` | the user query and its rewrite (simple mode) |
| `rag.retrieval.top_k` / `.min_score` / `.results_count` | retrieval params and outcome |
| `rag.retrieval.collections[]` | searched collection names |
| `rag.chunk.id` / `.score` / `.repo` / `.path` / `.line_range` | per-chunk attributes in the `chunk_retrieved` event |
| `rag.reranker.model` / `.top_n_before_rerank` | rerank stage, when used |

**`agent.*`** (dagstack-specific):

| Attribute | Semantic |
|---|---|
| `agent.pipeline` | `simple`, `agent`, `two_agent` |
| `agent.role` | `analyst`, `answerer`, `describer`, `planner`, `executor` |
| `agent.iteration` / `agent.iteration.max` | loop state |
| `agent.decision` | `continue`, `need_more`, `finish` plus a free-form `agent.decision.rationale` |

**`prompt.*`** (dagstack-specific, complementary to OTel GenAI):

| Attribute | Semantic |
|---|---|
| `prompt.template.id` / `.template.version` | prompt template lookup |
| `prompt.section.system.tokens` / `.user.tokens` / `.history.tokens` / `.tools.tokens` | per-section breakdown |
| `prompt.total_tokens` / `.token_budget` | total vs budget |
| `prompt.truncated` | boolean — whether history was truncated to fit the budget |
| `prompt.content.markdown` | the full assembled prompt (subject to truncation) |

**Correlation keys (baggage-propagated):** `session.id`, `conversation.id`, `agent.run.id` — for grouping all LogRecords of a single user interaction. Required in baggage for propagation across service boundaries (see §3.4).

##### 5.5.3. Typed events (`event.domain = "ai_agent"`)

Minimum set (the full schema lives in `_meta/events/ai_agent.yaml`):

| event.name | emit trigger | required attributes |
|---|---|---|
| `ai_agent.context.assembled` | the prompt is ready to be sent to the LLM | `prompt.total_tokens`, `prompt.token_budget` |
| `ai_agent.llm.request` | before an LLM call | `gen_ai.system`, `gen_ai.request.model`, `gen_ai.operation.name` |
| `ai_agent.llm.response` | after an LLM response | `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons` |
| `ai_agent.llm.retry` | retry attempt | `gen_ai.request.model`, `retry.attempt`, `retry.reason`, `retry.backoff_ms` |
| `ai_agent.tool.called` | tool dispatch | `gen_ai.tool.name`, `gen_ai.tool.call.id` |
| `ai_agent.tool.returned` | tool response | `gen_ai.tool.name`, `operation.duration_ms`, `operation.status` |
| `ai_agent.retrieval.requested` / `.completed` | RAG search boundary | `rag.retrieval.top_k`, `rag.retrieval.min_score` |
| `ai_agent.iteration.started` / `.completed` | agent-loop boundary | `agent.iteration`, `agent.pipeline`, `agent.role` |
| `ai_agent.decision` | pipeline state change | `agent.decision`, `agent.decision.rationale` |
| `ai_agent.session.started` / `.completed` | session lifecycle | `session.id`; on completed: `gen_ai.usage.input_tokens` sum, `.output_tokens` sum, `agent.iteration` max |

##### 5.5.4. Recommended span structure

```
span: ai_agent.session (root)
  └─ span: ai_agent.run (single user query)
      └─ span: ai_agent.iteration.1
          ├─ span: ai_agent.context.assembly
          ├─ span: gen_ai.chat (LLM call — OTel GenAI spec)
          │   events:  ai_agent.llm.request → ai_agent.llm.response
          ├─ span: mcp.tools/call (tool dispatch — OTel MCP spec)
          │   events:  ai_agent.tool.called → ai_agent.tool.returned
          └─ span: mcp.tools/call (read_file)
      └─ span: ai_agent.iteration.2 …
```

OTel-compliant spans with custom events. Traces-logs correlation is automatic (trace_id/span_id in LogRecord, §3.4).

##### 5.5.5. Body truncation and privacy

Prompts and LLM responses are often 100KB+, so the defaults are:

```yaml
logging:
  ai_agent:
    max_body_bytes: 4096            # truncate fields matching *.content / *.markdown
    capture_bodies: false           # production default: do not write prompt.content / response.content
    hash_bodies: true               # SHA-256 prefix (8 hex bytes) for dedup without storing the content
    body_hash_salt: ${BODY_HASH_SALT:-}  # optional salt so hashes do not leak content across deployments
```

**Truncation:** any attribute whose name matches `*.content`, `*.markdown`, `*.code`, `*.table_json`, or `*.html` is trimmed to `max_body_bytes` with `*_truncated: true` and `*_original_bytes: <N>` set. The binding provides a `truncate(value, max_bytes)` helper.

**Privacy mode** (`capture_bodies: false`): such fields are replaced with `""` or `null`, leaving only metadata attributes (`gen_ai.usage.*`, `*_original_bytes`, `*_hash`). Enable `capture_bodies: true` only in an explicit debug mode (e.g., via env `DAGSTACK_DEBUG_AI=true`).

**Hashing:** `*.content.hash = sha256(content)[:8]` lets you dedup and correlate identical prompts across runs without storing plaintext. The salt addresses environments where a hash leak is itself a risk.

#### 5.6. Packaging

All conventions live under `_meta/conventions/`:

- `_meta/conventions/operation_kinds.yaml` — enum extension for §5.1.
- `_meta/conventions/metadata_hints.yaml` — the full list of suffix hints from §5.4.
- `_meta/conventions/genai_otel_mapping.yaml` — the dagstack ↔ OTel GenAI mapping (§5.5.1).
- `_meta/events/<domain>.yaml` — per-domain event schemas (§5.2). `dagstack`, `progress`, `rag`, `agent`, `prompt`, `mcp`, `ai_agent` are reserved core domains.

Emitters (the §9.3 of config-spec pattern, adapted for logger-spec) generate per-language constants: Python enums + Literal types, TS `as const` + typed event signatures, Go const blocks. Golden fixtures in `conformance/events/` provide validation for bindings.

### 6. Scoped logger overrides

Use case: a single agent run needs to be logged into an InMemorySink for tests; a single endpoint MUST write to a separate file sink for audit; a single plugin hook MUST redact more than the default. The solution is a **scoped logger override** — temporarily replacing sinks / processors / min_severity for a limited execution scope.

#### 6.1. Scoped sink replacement

```
scoped_logger := logger.withSinks([InMemorySink(capacity=1000)]): Logger
scoped_logger := logger.appendSinks([SentrySink(dsn)]): Logger
scoped_logger := logger.withoutSinks(): Logger                      # drop all sinks — records go to /dev/null
```

`withSinks` is a full replacement; `appendSinks` adds extra sinks on top of those inherited from the parent; `withoutSinks` is discard mode. The returned logger is a child (§3.3); all inheritance rules apply.

#### 6.2. Scoped context manager

For a lexically bounded scope — context manager / resource block / defer pattern (binding-specific idiom):

```python
# Python
with logger.scopeSinks([InMemorySink()]) as scoped_logger:
    run_agent_pipeline(scoped_logger)
    # inside: all emits go to InMemorySink
# outside: normal sinks restored
```

```typescript
// TypeScript
await logger.withScopedSinks([new InMemorySink()], async (scoped) => {
    await runAgentPipeline(scoped);
});
```

```go
// Go
ctx, scoped := logger.WithScopedSinks(ctx, []Sink{NewInMemorySink()})
defer scoped.Close()
runAgentPipeline(ctx, scoped)
```

The binding chooses the idiom (with-statement / callback / ctx + defer); the semantics are the same: the scope only affects emits made through `scoped_logger` and its further children; the global `Logger.get(name)` keeps writing to the global sinks.

#### 6.3. Use cases

- **Tests**: `InMemorySink` for assertions on emitted records. A binding-level `CaptureLogs()` test fixture wraps `withSinks`.
- **Per-run audit**: a future dagstack-orchestrator can create a scoped logger per run that writes to a dedicated file plus pushes to the global OTLP. On run finalize, flush the scoped file.
- **Per-hook redaction**: a plugin hook handles a sensitive payload — a scoped logger with an augmented `RedactionProcessor`.
- **Debug session**: the user toggles `debug_capture_bodies` — a scope on a single request enables `capture_bodies: true` while other requests stay in privacy mode.

#### 6.4. Anti-patterns

- **Long-lived scoped loggers** — when an override outlives a single operation boundary, prefer creating a dedicated named logger via `Logger.get(name)` with its own config section. Scoped = ephemeral.
- **Scope leak through async** — a binding MUST guarantee that a scope does not outlive its enclosing block. For Python async, use `asynccontextmanager`; for TS, callback-scoped APIs; for Go, `ctx` cancellation. The binding ships a conformance test for scope isolation.

### 7. Sinks and adapters

A sink is a destination for LogRecords. The abstraction mirrors `ConfigSource` (`config-spec` §8) — a conventional `id`, an `emit()` method, optional async behavior, and `close()` for cleanup.

#### 7.1. Sink contract

```
Sink {
  id:            string                         # e.g., "console:dev-pretty", "otlp:https://otel.example:4318"
  emit(record: LogRecord): void                 # non-blocking; queues the record for delivery
  flush(timeout: Duration): FlushResult         # block until buffered records are delivered or the timeout fires
  close(): void                                 # flush + release resources
  supports_severity(severity_number): boolean   # optional filter hint; rejected severities are skipped early
}
```

**Non-blocking emit contract:** sinks MUST NOT block the caller of `logger.info(...)`. Delivery goes through an internal buffer plus a background worker; the overflow strategy is drop-oldest (default) or block (configurable per sink). Drop events are counted in `logger.metrics.dropped_total{sink_id}` (§12).

#### 7.2. Sink adapter roadmap

| Adapter | Purpose | Phase | Notes |
|---|---|---|---|
| `ConsoleSink(mode)` | stdout/stderr: JSON wire or pretty dev | 1 | auto-detects TTY for the default mode |
| `FileSink(path, rotation)` | local file with native rotation | 1 | size-based and time-based rotation |
| `InMemorySink(capacity)` | tests, ring-buffer | 1 | for conformance and application self-tests |
| `OTLPSink(endpoint, protocol)` | OTLP/gRPC or OTLP/HTTP | 2 | **primary production exporter** |
| `SyslogSink(transport, facility)` | BSD syslog / RFC 5424 | 2 | legacy integration |
| `SentrySink(dsn, min_severity)` | Sentry errors only | 2 | severity_number ≥ 17 (ERROR+); body and attrs become a Sentry event |
| `LokiSink(endpoint)` | Grafana Loki `/loki/api/v1/push` | 2 | when no OTel collector is available |
| `FluentBitForwardSink(host)` | Fluent Bit Forward protocol | 2 | a common sidecar pattern |
| `CloudWatchLogsSink(group, stream)` | AWS CloudWatch Logs | 3 | IAM auth, batch put |
| `GCPCloudLoggingSink(project, log)` | Google Cloud Logging | 3 | OAuth/service account |
| `KafkaSink(brokers, topic)` | Kafka producer | 3 | high-throughput pipelines |
| `ElasticsearchSink(hosts, index)` | Elasticsearch bulk API | 3 | consider routing via Logstash instead |

**Primary production path**: `OTLPSink` → OTel Collector → any backend. `LokiSink` / `SentrySink` / cloud-specific sinks are for environments without a collector.

**URI-style `id` convention**: `otlp://host:port`, `loki://host`, `file:///var/log/app.jsonl` — for diagnostics.

#### 7.3. Multi-sink routing

A logger can be configured with multiple sinks plus a per-sink severity filter:

```yaml
logging:
  sinks:
    - id: console:dev
      type: console
      mode: pretty
      min_severity: DEBUG
    - id: file:app
      type: file
      path: /var/log/app.jsonl
      rotation: { size: 100MB, keep: 10 }
      min_severity: INFO
    - id: otlp:prod
      type: otlp
      endpoint: https://otel.example.com:4318
      protocol: http/protobuf
      min_severity: INFO
    - id: sentry:errors
      type: sentry
      dsn: ${SENTRY_DSN}
      min_severity: ERROR
```

A LogRecord is emitted into every sink whose filter passes. Sinks are **independent** — a failure of one does not affect the others.

#### 7.4. Self-observability isolation: `dagstack.logger.internal`

The logger needs a diagnostic channel for its own errors (sink failures, buffer overflow, malformed records, reconfigure failures). The reserved named logger is **`dagstack.logger.internal`**.

**Isolation contract (normative):**

- `dagstack.logger.internal` **MUST NOT** inherit sinks from the root logger by default. This prevents an infinite loop: if the OTLPSink fails, the diagnostic is written through `dagstack.logger.internal`, which has the same OTLPSink, which fails, and so on.
- A binding **MUST** configure for `dagstack.logger.internal` a minimal dedicated sink — stderr JSON-lines (direct write, no background worker / batching) — so that the diagnostic reaches the operator even when the main logger pipeline is broken.
- An application **MAY** override the sink for `dagstack.logger.internal` through an explicit config (`logging.loggers["dagstack.logger.internal"].sinks`), but receives a warning if the configuration includes a sink that is also in the root logger chain.
- Records in `dagstack.logger.internal` **NEVER** pass through the LogProcessor chain (§8) — no redaction, no sampling — to keep a minimal delivery guarantee for the diagnostic.

**Use cases for emitting to `dagstack.logger.internal`:**

- Sink errors, buffer overflow, dropped records.
- Reconfigure rejections (§9.3).
- Schema validation failures for typed events (§5.2).
- Async callback errors (e.g., when a scoped override leaks across an async boundary, §6.4).

### 8. LogProcessor chain (Phase 2)

A middleware pipeline between the logger API and the sinks:

```
processor: (LogRecord) -> LogRecord | null    # null = drop
```

The chain is applied in the config-listed order. Use cases:

- **Redaction** — mask attribute values matching a pattern (§10).
- **Sampling** — rate-based / probability-based drop (§11).
- **Enrichment** — add resource attributes, session_id, a feature_flags snapshot.
- **Filtering** — drop records by predicate (e.g., a noisy third-party logger).

**Phase 1** does not require processors — the minimum viable logger has no chain. Phase 2 introduces a standardized `_meta/processors.yaml` registry (redaction, sampler_rate, sampler_trace_ratio) plus emitters.

### 9. Configuration via config-spec

The logger is a **consumer** of `dagstack/config-spec`, not a standalone config loader. The logger section in YAML (`logging.*`) is part of the application config.

#### 9.1. Canonical `logging:` section

```yaml
logging:
  # Root logger defaults
  level: ${LOG_LEVEL:-INFO}
  resource:
    service.name: ${SERVICE_NAME:-pilot-app}
    service.version: ${SERVICE_VERSION:-dev}
    deployment.environment: ${DAGSTACK_ENV:-development}

  # Per-logger level overrides (dot-hierarchy)
  loggers:
    dagstack.rag.retriever: DEBUG
    httpx: WARN
    urllib3: WARN

  # Sinks (§7.3)
  sinks:
    - id: console
      type: console
      mode: ${LOG_CONSOLE_MODE:-auto}     # auto | pretty | json
      min_severity: ${LOG_LEVEL:-INFO}

    - id: otlp
      type: otlp
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT}
      protocol: http/protobuf
      min_severity: INFO
      enabled: ${OTLP_ENABLED:-false}

  # Phase 1 redaction config (§10.4) — additive suffix-based masking
  redaction:
    extra_suffixes:    []                # additive to the base set; lowercase ASCII, leading underscore
    replace_defaults:  false             # when true, base set is NOT applied (drastic — see §10.4 warning)

  # Processor chain (Phase 2 — regex-based processors layered on top of §10.4)
  processors:
    - type: redaction
      extra_patterns: []                 # regex patterns; complements redaction.extra_suffixes
    - type: sampler_rate
      applies_to_severity_below: INFO    # sample DEBUG/TRACE only
      rate: 0.1
```

YAML keys are snake_case (`extra_suffixes`, `replace_defaults`); per-language bindings convert to their idiomatic naming (`extraSuffixes` in TypeScript, `ExtraSuffixes` in Go) at deserialisation. The two-axis split — Phase 1 `redaction:` for fast suffix matching, Phase 2 `processors:` for regex/value-pattern policies — keeps the hot path branch-free.

#### 9.2. Bootstrap API

```
logger_config := config.getSection("logging", LoggerSchema)
Logger.configure(logger_config)              # global bootstrap
logger := Logger.get("dagstack.rag.retriever", version)
```

`LoggerSchema` is published as part of logger-spec conformance metadata (`_meta/logging_section_schema.yaml`). Bindings emit a native schema (Pydantic, zod, Go struct) through the standard emitter.

#### 9.3. Runtime reconfiguration

```
config.onSectionChange("logging", LoggerSchema, (old, new) -> {
    Logger.reconfigure(new)
})
```

`Logger.reconfigure(new)` MUST:

1. Validate the new config (sinks addressable, levels valid).
2. Atomic swap — new records go to the new configuration; in-flight records are flushed through the old sinks before close.
3. Closing old sinks: previous sinks that do not survive into the new config get `flush(timeout)` → `close()` async.
4. Adding new sinks: eager init (connect to the OTLP endpoint, open the file) — if it fails, the reconfigure is rejected and the old config keeps active (parallel to `config-spec` §8.4 validation rollback).

### 10. Redaction

#### 10.1. Default masking

The logger automatically masks values of `attributes[k]` where `k` matches (case-insensitive) one of:

- **Base patterns** (reusing `config-spec/_meta/secret_patterns.yaml`): `*_KEY`, `*_SECRET`, `*_TOKEN`, `*_PASSWORD`, `*_PASSPHRASE`, `*_CREDENTIALS`.
- **Extension patterns** — Phase 1 public API per §10.4; Phase 2 ships richer regex-based processors (§10.3).

The mask is rendered as the literal `"***"` (sink-side; wire-format outputs `"***"` rather than the original value). The raw value **never** crosses the logger API boundary in attributes flagged as secret.

The **body** (LogRecord.body) **is not redacted automatically** — developers are expected to format the body without secrets. Redacting an arbitrary substring in the body would require pattern matching on plaintext (expensive, brittle). The spec may enable an opt-in body-scanning processor in Phase 2+.

#### 10.2. Nested attributes

For `Map<string, Value>` inside attributes, redaction applies **recursively** to keys at every level. `attributes = { user: { api_key: "…" } }` → `attributes = { user: { api_key: "***" } }`.

Inside lists (`List<Value>` per §1), each element is recursed into when it is a `Map<string, Value>` — a secret-matching key inside any list element is masked, even when the list-carrier key itself is not secret. `attributes = { events: [ { api_key: "…" } ] }` → `attributes = { events: [ { api_key: "***" } ] }`. Heterogeneous lists are supported: primitive elements alongside maps pass through unchanged. When a secret-matching key carries a list or map value, the **whole value** is replaced with the placeholder (whole-value mask wins; recursion does not re-enter masked subtrees).

#### 10.3. Custom redactors (Phase 2)

An application can add a custom `RedactionProcessor`:

```yaml
processors:
  - type: redaction
    extra_patterns:
      - pattern: "user\\.email"
        replacement: "<redacted-email>"
      - pattern: ".*_pii"
        replacement: "***"
```

Regex syntax is the ECMA-262 subset (the same as in `config-spec/_meta/interpolation.yaml` — we reuse the formal definition).

#### 10.4. Phase 1 public API for extra suffixes

A binding **MUST** expose a public configuration option that lets the application register additional secret suffixes at bootstrap, without waiting for the Phase 2 processor pipeline.

The shape of the option (per-language naming follows §3.5 idioms):

```
RedactionConfig := {
  extra_suffixes?:    list<string>      # default: empty — additive to base
  replace_defaults?:  bool              # default: false — when true, base suffixes are NOT applied
}

configure(
  ...
  redaction?: RedactionConfig
)
```

Semantics:

- **Additive by default.** With `extra_suffixes = ["_apikey", "_x_internal_token"]` and `replace_defaults = false` (or unset), the effective set is `BASE ∪ extra_suffixes`.
- **Replace mode.** `replace_defaults = true` swaps the base set for the supplied list — for the rare case when a deployment intentionally narrows the safety net.
- **Validation.** At `configure()` time, a binding MUST reject suffixes that are empty strings, contain whitespace, or are not lowercase ASCII. A binding MAY defer this validation **only** when its public configure surface is option-builder–style (functional options, fluent builders) and option construction is documented as side-effect-free; in that case the validation MUST surface no later than the first `emit()` call after `configure()` returns. After the first emit, subsequent emits MUST NOT silently re-validate. Document the chosen mode in the per-binding ADR.
- **Disable-all warning.** A binding **SHOULD** emit a `WARN`-severity diagnostic on the `dagstack.logger.internal` channel (§7.4) when `configure()` applies `replace_defaults = true` together with an empty `extra_suffixes` list, since this disables all suffix-based masking. The diagnostic MAY be suppressed via a binding-specific opt-out for the rare deployment that intentionally relies on Phase 2 processor-based redaction (§10.3) only.

A binding MUST keep the base set immutable across the process lifetime and SHOULD expose it as a public read-only constant (e.g., `DEFAULT_SECRET_SUFFIXES`) so tests can build extension lists relative to it. The base set is an opinionated 6-element subset of `config-spec/_meta/secret_patterns.yaml` covering the most universal credential suffixes; the explicit list is fixed at v1.1 to preserve API stability:

```
_KEY, _SECRET, _TOKEN, _PASSWORD, _PASSPHRASE, _CREDENTIALS
```

Prefix and exact-match entries from `secret_patterns.yaml` are out of scope for the Phase 1 logger API; richer matchers ship via the Phase 2 processor pipeline (§10.3). Bindings MUST also expose an `EffectiveSecretSuffixes` accessor (per-language naming follows §3.5) that returns the post-`configure()` set, so test fixtures and the disable-all diagnostic in §7.4 can introspect the active configuration.

**Conformance fixture.** `_meta/fixtures/redaction_extra_suffixes.json` exercises the additive and replace modes (each with its own input attributes), asserting that the post-redaction wire output matches across bindings.

**Why a dedicated `RedactionConfig` instead of a plain string list.** A dedicated struct leaves room for future additive fields (`replacement: string`, `key_prefixes: list[string]`, `value_patterns: list[regex]`) without breaking the v1.x API; a bare `extra_suffixes: list[string]` would force a breaking change when richer policies (Phase 2) get added.

### 11. Sampling

#### 11.1. Phase 1 — severity-based filter

Per-logger and per-sink `min_severity` (§7.3, §9.1). Records below the threshold **are not emitted** — early drop, no allocations.

#### 11.2. Phase 2 — processor-based samplers

```yaml
processors:
  - type: sampler_rate
    applies_to_severity_below: INFO
    rate: 0.1                 # 10% of TRACE/DEBUG records pass
  - type: sampler_trace_ratio
    preserve_sampled_traces: true    # if a trace is OTel-sampled, always emit; otherwise drop by rate
    rate: 0.05
```

- **`sampler_rate`** — simple probability drop (deterministic hash plus rate).
- **`sampler_trace_ratio`** — trace-correlated: when the active span has `trace_flags.sampled = true`, the record is always emitted; otherwise it is rate-based dropped. Preserves logs for sampled traces (a standard OTel practice).

#### 11.3. Tail-based sampling — out of scope

Tail-based sampling (batching records to decide on samples after a full trace completes) requires Collector/backend-side logic. Delegated to the OTel Collector; **not part** of the logger-spec contract.

### 12. Observability of the logger itself

The logger exposes self-metrics (Phase 1 — optional; Phase 2 — mandatory):

```
logger.metrics = {
  records_emitted_total{logger_name, severity}: counter,
  records_dropped_total{sink_id, reason}: counter,    # reason: buffer_overflow, sink_error, filtered
  sink_flush_duration_seconds{sink_id}: histogram,
  sink_errors_total{sink_id, error_type}: counter,
  reconfigure_total{result}: counter,                 # result: ok, rejected, error
  active_loggers_gauge: gauge,
  buffer_depth{sink_id}: gauge,
}
```

The binding picks the publishing idiom (Python OTel Metrics SDK, Go `expvar` or Prometheus, TS `prom-client`). Naming follows OTel semantic conventions (the `otel.logger.*` prefix).

### 13. Async and shutdown

**Non-blocking emit** — §3.2 / §7.1. Contract recap:

- `logger.info(...)` returns immediately. The internal buffer batches records per sink; a worker thread / async task flushes them.
- The caller never waits for network I/O (OTLP push, Loki push, SyslogSink UDP send — all batched).
- The overflow strategy is configurable per sink: `drop_oldest` (default), `drop_newest`, `block` (for tests / critical audit logs).

**Shutdown protocol:**

```
logger.flush(timeout: Duration): FlushResult
logger.close(): void        # = flush(default_timeout) + release resources
```

`FlushResult { success: bool, partial: bool, failed_sinks: [{sink_id, error}] }`. An application SHOULD call `logger.close()` in a shutdown hook. If the process exits without flushing, buffered records are lost (a warning to the diagnostic channel at close-detected time when possible).

**Signal handling** is binding-specific. Recommended: the logger registers atexit / SIGTERM handlers (Python / Go); for TS the recommendation is an explicit `await logger.close()` in graceful shutdown.

## Conformance

A binding is **conformant with logger-spec v1.0** when it passes the four
test categories below. The list is normative; specific golden fixtures and
the runner CLI are distributed via the spec repo (`conformance/` —
empty in v1.0, populated incrementally during the v1.0 → v1.1 phase as
the first three bindings stabilise). Until `conformance/` ships, a
binding MUST cover the same surface with native tests and document the
mapping in its `tests/conformance/README.md`.

### 14.1. Wire-format roundtrip

For each wire format declared in §3 (OTLP protobuf, OTLP JSON, dagstack
JSON-lines), a binding MUST:

- **Emit → parse → re-emit** a fixed set of representative LogRecords
  (one per severity, one per primary `event.domain`, one with redaction
  applied, one with full context).
- **Compare** the second emit byte-for-byte against the first emit
  (Canonical JSON rules from `config-spec` §9.1.1 apply for the
  dagstack JSON-lines format).
- **Reject** the parse step if a non-conformant input is fed (unknown
  severity_number, missing instrumentation_scope, malformed trace_id).

The conformance fixtures distribute golden inputs and expected outputs
under `conformance/wire/{otlp_proto, otlp_json, dagstack_jsonl}/`.

### 14.2. Context propagation

A binding MUST demonstrate, by test, that:

- `trace_id` / `span_id` / `trace_flags` extracted from the OTel Context
  API (§3.4) appear unchanged in every LogRecord emitted while that
  context is active.
- A nested context overrides its parent for the duration of the inner
  scope and restores the parent on exit.
- A LogRecord emitted outside any context omits `trace_id` / `span_id`
  rather than synthesising zeros.

A binding MUST also pass `_meta/fixtures/trace_context_propagation.json`
(per spec §3.4.4): the cases that match the binding's modes
(`auto-inject` and/or `explicit-ctx`) MUST produce wire bytes whose
`trace_id` / `span_id` / `trace_flags` fields equal the fixture's
`expected` block, with the sample IDs substituted at test time. Cases
asserting `fields_absent` MUST verify those keys are missing from the
emitted wire (not present-but-zero).

A binding's default value of `auto_inject_trace_context` **MUST** match
the value declared in its per-language ADR (§3.5). A test harness
verifies the default at startup before running fixture cases; a binding
that ships a different default than its ADR declares is non-conformant.

### 14.3. Semantic conventions

For the conventions in §5 (operations, typed events, progress) and the
AI-agent extension pack (§5.5), a binding MUST:

- Validate every LogRecord against the relevant schema entry from
  `_meta/conventions/*.yaml` (the source of truth for typed-event
  attributes; v1.0 ships at minimum
  `_meta/conventions/genai_otel_mapping.yaml` and
  `_meta/conventions/operations.yaml`).
- Reject `event.name` values whose attribute set does not match the
  schema, with a `ConfigError`-shaped exception listing the missing or
  surplus attributes.
- Pass the `prompt.body` / `tool.input.body` redaction default
  (`capture_bodies: false`) test from `conformance/redaction/`.

### 14.4. Sinks Phase 1

The three Phase 1 sinks (§7.1 ConsoleSink, FileSink, InMemorySink) MUST
each pass:

- A non-blocking emit test under load (10 000 records emitted in under
  one second on the binding's reference platform; the caller is never
  blocked for longer than the configured back-pressure threshold).
- A drop-on-overflow accounting test (the
  `logger.metrics.dropped_total{sink_id}` counter increments by exactly
  the number of dropped records).
- A close / flush test (an explicit `logger.close()` flushes pending
  records before returning; an unflushed shutdown emits a warning to
  the diagnostic channel `dagstack.logger.internal`).

### 14.5. Source-of-truth metadata

Bindings MUST consume `_meta/` from the spec repo (vendored or as a git
submodule) rather than hand-coding the values. The v1.0 set is:

- `_meta/severity.yaml` — the 1–24 severity_number table and the six
  canonical `severity_text` strings.
- `_meta/event_domains.yaml` — the registered values of `event.domain`
  (`operation`, `progress`, `agent`, `rag`, `prompt`, …) with their
  required attribute sets.
- `_meta/conventions/*.yaml` — typed-event schemas for §5 and §5.5
  (one file per convention family).
- `_meta/canonical_json.yaml` — re-export from `config-spec`; logger-spec
  does not re-define Canonical JSON, only references it.
- `_meta/fixtures/*.json` — cross-binding parity fixtures (input + config
  + expected output triples). v1.1 ships
  `_meta/fixtures/redaction_extra_suffixes.json` for §10.4; future
  fixtures (e.g., `redaction_nested_attributes.json` for §10.2) join
  this family without further spec amendments.

A binding that hand-codes severity tables or event-domain attribute sets
is **not** conformant — drift between the spec and the binding will
surface only at production divergence and is exactly the failure mode
the `_meta/` source of truth eliminates.

### 14.6. Redaction (§10)

A binding MUST pass `_meta/fixtures/redaction_extra_suffixes.json`:

- For each `cases[]` entry, applying `config.redaction` at `configure()`
  and emitting a record with `attributes = case.input`, the
  post-redaction wire output MUST equal `case.expected` byte-for-byte
  (per the binding's default wire format; Canonical JSON rules apply for
  dagstack JSON-lines).
- For each `validation_failures[]` entry, `configure()` (or the first
  `emit()` after configure when validation is deferred per §10.4) MUST
  raise an error whose message contains the fixture's `expected_error`
  string — case-insensitive substring match per the fixture's
  `_validation_error_match` field. Bindings are free to wrap, prefix, or
  localise the error provided that the substring appears verbatim.
- The §10.2 list-of-map traversal and the whole-value-mask-wins
  invariant are also covered by the same fixture
  (`additive_recurses_into_list_of_maps`,
  `additive_recurses_into_nested_dict`).

### 14.7. Phase 1 minimum vs full conformance

The four sub-sections above describe **full v1.0 conformance**. A
binding MAY publish under a `phase1-partial` tag if it passes 14.1
(dagstack JSON-lines only — OTLP wires deferred), 14.2, the §5
operations subset of 14.3 (extension-pack §5.5 deferred), and 14.4.
Such a binding must document the gap explicitly in its README so a
consumer can choose between `phase1-partial` (Phase 1 sinks ready) and
`v1.0` (full conformance).

## Consequences

### Positive

- **Cross-language parity** — the operator sees a single log model; log pipelines are unified.
- **OTel-compatible wire** — a direct path to any observability backend through OTLP, with no lock-in.
- **Config-driven** — through `config-spec`, a single model for ops: reconfiguration of level and sinks without a restart.
- **Automatic trace-log correlation** — mandatory context propagation makes the `trace_id ⇄ log` link the default.
- **Unified redaction** — a single pattern list (`secret_patterns.yaml`) reused between config-spec and logger-spec.
- **Rich semantic conventions** (§5) — operations/events/progress/metadata-hints plus the AI-agent extension pack deliver Dagster-class observability UI without breaking the OTel wire.
- **AI/LLM observability out of the box** — §5.5 makes a pilot consumer and future agentic applications first-class: token accounting, prompt-assembly tracing, tool-call dispatch, retries, session correlation — standardized rather than ad-hoc.
- **OTel GenAI conformance** — we do not invent a parallel standard; any OTel-compatible observability platform (Datadog, Grafana Cloud, Honeycomb, Langfuse) reads our logs natively.
- **Scoped overrides** (§6) — test isolation, per-run audit, and debug capture without breaking the global logger config.

### Negative

- **Dependency on the OTel SDK** in every binding — operational overhead for minimal applications.
- **Dependency on `config-spec`** — the logger is not self-contained, and even simple use cases require a config-spec binding. Mitigation: the binding can offer `Logger.minimalBootstrap()` with hardcoded defaults for CLI scripts without a config file.
- **Async complexity** — non-blocking emit requires lifecycle management (flush, close), which new developers often forget.
- **Per-language SDK divergence risk** — the OTel SDK between Python/TS/Go has detail differences (context API shape, exporter lifecycle); conformance fixtures are critical for catching drift.
- **OTel GenAI is a moving target** — the spec fluctuates between draft and stable; the conformance commitment (§5.5.1) requires active tracking and syncing on major logger-spec releases. Risk: breaking changes in OTel will require deprecation cycles.
- **Event schema maintenance** — `_meta/events/*.yaml` grows with the number of domains (rag, agent, prompt, ai_agent, mcp); schema evolution (adding/removing attributes) requires disciplined versioning (`event.schema_version` + conformance fixtures per version).

## Explicitly out of scope

### Deferred to Phase 2+

- `OTLPSink`, `LokiSink`, `SentrySink`, `SyslogSink`, `FluentBitForwardSink` — §7.2 Phase 2.
- Cloud-specific sinks (CloudWatch, GCP Logging, Kafka, Elasticsearch) — §7.2 Phase 3.
- The LogProcessor chain (§8), including rate / trace-ratio samplers — Phase 2.
- A custom `RedactionProcessor` with user patterns — Phase 2.
- Logger self-metrics (§12) — optional in Phase 1, mandatory in Phase 2.

### Forever out of scope

- **Tracing and Metrics SDK** — use OTel Tracing / OTel Metrics directly. logger-spec covers logs only, with a hand-off API for correlation.
- **Tail-based sampling** — a Collector/backend concern (§11.3).
- **Body pattern scanning** (regex over LogRecord.body for catch-all secret detection) — expensive, brittle. Developers shape the body without secrets; enforcement is via code review and static analysis.
- **Log-based alerting rules** — a backend concern (Grafana Alerting, CloudWatch Alarms).
- **Multi-tenant log isolation** — an infrastructure concern (separate OTel pipelines per tenant); logger-spec emits the `tenant.id` attribute and leaves routing to the platform layer.

## v1.1 backlog (post-pilot)

From the architect must-fix round 2026-04-19 (should-consider):

1. **Structure §5 split** — move the AI-agent extension pack (§5.5) out into a separate ADR-0002, leaving the core §5 at roughly 1500 lines (operations / typed events / metadata hints / progress). Non-AI consumers do not pull AI specifics into their mental model.
2. **OTel GenAI version pinning policy** (§5.5.1) — pin "logger-spec v1.x conforms to OTel GenAI semconv vX.Y.Z"; a dual-emit flag `gen_ai_compat_mode: v1 | v2 | both` in config for migration windows. Otherwise the §5.5.1 commitment is undeliverable (OTel has already renamed `gen_ai.completion.*` → `gen_ai.response.*` and `prompt_tokens` → `input_tokens` between releases).
3. **Reconfigure atomicity model** (§9.3) — explicitly fix snapshot-per-emit as the default (a new config is not retroactive for in-flight records); shared-buffer swap is optional with documented double-delivery windows. A timeout budget on the reconfigure as a whole.
4. **Per-request body capture** (§5.5.5) — the scoped-logger pattern as the primary mechanism (FastAPI middleware / TS middleware keyed by correlation-id / debug-header with ACL), with the env var `DAGSTACK_DEBUG_AI` as a fallback for whole-process debug. Example code in ADR-0002.
5. **Event emitters scope-down** (§5.6) — generate only `EVENT_DOMAIN_NAME` constants; `required_attributes` / `optional_attributes` stay in YAML for docs and conformance tests only. This reduces emitter complexity by an order of magnitude.
6. **Sampler interaction with scoped loggers** (§6.1) — `withSinks` / `scopeSinks` reset sampling processors; `appendSinks` preserves them. Fix this explicitly.
7. **Progress-sink deprecation path** (§5.3 of this spec ↔ ADR-0003 §6 of plugin-system-spec) — a migration ADR in `plugin-system-spec`: bindings implement a compat shim (the old Progress sink API emits `event.domain=progress` events), to be removed in logger-spec v2.x.

Pilot-driven:

8. **OTLPSink** — a minimum set for the start of Phase 2. MVP: OTLP/HTTP protobuf export with batching (1s flush or 512 records).
9. **Sentry SDK bridge** — `SentrySink` for error-severity. Integration tests against the Sentry SaaS.
10. **Logger self-metrics** — graduate from optional Phase 1 to mandatory. OTel Metrics SDK integration. Phase 1 gap: `FlushResult.dropped_count` as a fallback when metrics are disabled (§13 shutdown recap).
11. **LogProcessor chain** with redaction plus sampling — the core processor set.
12. **Exception stacktrace formatting convention** — `_meta/conventions/exception.yaml` with per-language guidance (Python traceback vs Go stacktrace vs JS stack) to reduce drift between bindings.
13. **OTel GenAI tracking** — automation to update `_meta/conventions/genai_otel_mapping.yaml` on every OTel GenAI semconv release; drift diagnostics between our conventions and upstream.
14. **Cost-tracking processor** (§5.5.2) — derive `gen_ai.usage.cost_usd` from `input_tokens`/`output_tokens` plus a pricing table per `gen_ai.system` and `gen_ai.request.model`. The pricing table is a separately distributed spec artifact, updated on demand.
15. **Bodies storage pointer pattern** — when `capture_bodies: false` but debugging is needed: write `*.content.ref` (a URI to side-channel storage — S3/GCS/local ring buffer); backends fetch the content on demand.
16. **AI-agent events — richer schemas** — `ai_agent.decision.rationale` is currently free-form text; v1.1 introduces a structured rationale (referenced chunks, confidence scores, alternative options considered).
17. **Progress events semantics** (§5.3) — agent-style `eta_ms` computation requires rate smoothing; a reference implementation in `_meta/` or binding-specific.
18. **`*.code.lang` pairing rule** (§5.4) — fix this for every hint suffix: `foo.code = "<code>"` plus `foo.code.lang = "python"` as two attributes, or a nested map. Pick one and document it.

## Open questions

1. **Async emit failure visibility** — when an OTLPSink fails on a background worker (a network error after the batch is assembled), is there application-level observability? Candidate: log via the `dagstack.logger.internal` named logger to a console sink (a cyclic risk if the console also fails). Bounded queue plus drop counter is the minimum viable answer.
2. **Plugin isolation of loggers** — plugin A writes to `dagstack.plugin_system.plugin_a`, plugin B writes to `dagstack.plugin_system.plugin_b`. Can plugin A read plugin B's log records via a hierarchical `Logger.get("dagstack.plugin_system")`? Current intuition: **no, write-only**; there is no read API exposed. Fix this explicitly.
3. **Exception formatting** — stack-trace unification across languages (Python traceback vs Go runtime.Callers vs JS stack). Adopt the OTel semantic conventions (`exception.stacktrace` as an opaque string), but pin the formatting convention in `_meta/`.
4. **Logger shutdown in a Python async context** — `atexit` does not cover async tasks. Solution: the binding documents the pattern `try/finally: await logger.close()` in a FastAPI lifespan. Is this an upper-bound spec mandate or binding-specific guidance?
5. **Resource detector set in Phase 1** — at a minimum `service.name`, `service.version`, `deployment.environment`, `process.pid`. More (k8s pod / aws instance metadata) — via OTel Resource Detectors in Phase 2.
6. **Event schema evolution policy** — when `_meta/events/<domain>.yaml` changes (a required_attribute added): breaking or non-breaking? Proposed rule: adding optional is minor; adding required or removing is a major bump of `event.schema_version`, with the old version supported for N releases. Formalize.
7. **Reserved vs user-defined namespaces** (§5.2 reserved domains) — strict validation in bindings (reject `logger.emitEvent("dagstack.foo", ...)` from non-dagstack-core code)? Or convention-level (warn, do not reject)?
8. **Scope propagation across async boundaries** — §6.4 Anti-patterns raises this. For Python `contextvars` works; for TS — `AsyncLocalStorage`; for Go — `context.Context`. Binding-specific, but conformance fixtures must cover the edge cases (async leak, thread pool, worker task).
9. **AI-agent correlation ID hierarchy** — `session.id` ⊃ `conversation.id` ⊃ `agent.run.id`? Or flat? A consumer business concern; the spec does not mandate but recommends hierarchical.
10. **`gen_ai.prompt.<N>` vs `prompt.content`** — OTel uses numbered `gen_ai.prompt.0.role` / `.content`; we added `prompt.content.markdown` for the assembled full prompt. Where do they overlap and which to use when? Proposal: the OTel-style is for structured conversational messages (role-segmented); our `prompt.content.markdown` is for the final assembled prompt text before tokenization. Clarify in `_meta/conventions/genai_otel_mapping.yaml`.
