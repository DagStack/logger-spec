# logger-spec

Language-agnostic spec for dagstack logger — OTel-compatible structured logging, context propagation, sinks, sampling, redaction.

Per-language bindings live in separate repos: `dagstack/logger-python`, `dagstack/logger-typescript`, `dagstack/logger-go`.

## Highlights

- **Wire format** — OTel Log Data Model v1.24 compatible (severity, body, attributes, resource, instrumentation_scope, trace/span ids).
- **Semantic conventions** (§5) — operations, typed events with a schema registry, progress events, metadata type hints for rich UI rendering.
- **AI-agent extension pack** (§5.5) — first-class observability for LLM/agent pipelines; conformant with [OTel GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) (including MCP conventions), extended with `rag.*` / `agent.*` / `prompt.*` namespaces.
- **Sinks and adapters** (§7) — Phase 1: console, file, in-memory; Phase 2+: OTLP, Syslog, Loki, Sentry, cloud-specific.
- **Scoped logger overrides** (§6) — per-run sink replacement for tests, audit, and debug without breaking the global config.
- **Configured via `dagstack/config-spec`** — unified YAML section `logging:` with runtime reconfiguration through `onSectionChange`.

## Scope

- Application logs (first-class). Spans/metrics — hand-off API for interop with the OTel SDK; full-fledged tracing/metrics are out of scope (use the OTel Tracing / Metrics SDK directly).
- Dagster-style event-tree richness is implemented as a convention over `attributes`, not as a new structure.

See `adr/0001-logger-contract.md`.

## Related

- [`dagstack/config-spec`](https://github.com/dagstack/config-spec) — config loader + runtime reconfiguration (the logger reads its config through this).
- [`dagstack/plugin-system-spec`](https://github.com/dagstack/plugin-system-spec) — plugin architecture (the logger's instrumentation_scope is associated with a plugin manifest; the Progress sink in ADR-0003 §6 is absorbed as the §5.3 convention of this spec).
