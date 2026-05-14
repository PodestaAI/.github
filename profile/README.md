# Podesta AI

**AI for non-tech industries.** Your domain experts deliver examples of the tasks they do — input and output — and our system builds a benchmark, generates a multimodal, multi-agent workflow against it, iterates until it passes, and deploys.

No prompt engineers required.

---

## What we do

Most AI tooling assumes the customer has prompt engineers on staff. That means tech companies get AI; everyone else gets demos. Podesta AI exists to close that gap — so a law firm, a manufacturer, an insurer, or a hospital can ship real AI workflows on the same footing as a Silicon Valley startup.

The mechanic:

1. A domain expert provides examples of a task they already do (input, output).
2. Akribes — our AI Workflow DSL — builds a benchmark from those examples.
3. The generator emits a multimodal, multi-agent workflow.
4. The workflow iterates against the benchmark until it passes.
5. It ships into production.

The result is an AI workflow that does the expert's task, validated against examples the expert themselves provided.

## Who it's for

We are not a vertical AI play. The wedge is a functional pattern:

> A team of 5 to 50 experts doing **repetitive high-stakes review or classification work**, 20 to 200 times a day, where humans currently produce the correct decisions and the bottleneck is throughput.

When that pattern is present — legal review, RMA triage, fraud triage, insurance claims, KYC/AML, compliance audit, medical intake, benefit eligibility — the examples-in, workflow-out approach works. The steady-state runs autonomously; ambiguous cases route to a human.

## Akribes — the AI Workflow DSL

Akribes is the language generated workflows are expressed in, and the runtime that executes them. It is what makes Podesta workflows portable, auditable, and re-trainable across model generations.

### What an `.akr` program looks like

A workflow is a self-contained typed program with declarative blocks for types, agents, tasks, flows, and the workflow itself. The compiler validates types and constraints before any LLM call.

```akribes
type Severity = "low" | "medium" | "high"

agent Triager:
  model: anthropic.opus_4_7
  thinking: true
  system: "You triage incident tickets without speculation."

task triage(ticket: str) -> Severity:
  actor: Triager
  prompt: "Triage {ticket}."
  rules:
    no_speculation "Do not speculate about facts the ticket does not state."
    cite_evidence  "Cite the passage that drove your verdict."
  validation_retries: 3
```

### Types validated before any LLM call

A structural type system with primitives (`str`, `int`, `float`, `bool`, `markdown`, `document`, `json`), collections (`list[T]`), choice types (`"a" | "b" | "c"` becomes a JSON-schema enum that the provider refuses to violate at generation time), and type aliases. Documents are uploaded once and converted to markdown via Docling or a vision model. The compiler rejects bad workflows before they make a single API call.

### Constraints enforced three ways

Every constraint on a field renders simultaneously into:

1. The **JSON schema** sent to the provider — structured output refuses non-conforming bytes at generation time.
2. The **prompt** — the model sees the rule under `### Constraints`, so it knows what the engine will enforce.
3. A **runtime Rust validator** — post-parse double-check; failures drive corrective retries.

Recognized phrases include `between A and B`, `matches /<regex>/`, `at least N words`, `non-empty`, `sorted by <field>`, and `validate_with: <fn>` for custom Rust validators. Every failure carries a stable diagnostic code (`AKRIBES-E-CONSTRAINT-*`) that survives prose revisions.

### Retry, parallelism, and composition

Three orthogonal retry knobs cover different failure modes:

| Knob | Default | Covers |
|------|---------|--------|
| `on_error: retry(N)` | none | Network errors, rate limits, provider 5xx |
| `validation_retries: N` | 2 | Schema and constraint-validator failures; corrective re-prompts |
| `workflow_retries` | 2 | Author-raised `fail` arms; re-runs the failing node only |

Workflows compile to a DAG. Independent tasks run in parallel automatically. `for X in list: max_concurrent: N` does parallel fan-out; `match` dispatches on union types; `call(...)` invokes another `.akr` script as a sub-execution with its own checkpoint state and validation budget.

### Editor support and SDKs

`akribes-lsp` implements the Language Server Protocol — diagnostics, completions, hover, go-to-definition, find-references, project-wide rename, and workspace symbols in VS Code, Sublime, or any LSP-compatible editor.

First-class SDKs in **TypeScript, Python, and Rust**, all supporting streamed output, channel selection (production / staging / draft), breakpoint-line debugging, eval execution, and a `triggered_by` audit field. Authentication is two-tier: long-lived service tokens for system integrations, scoped `akribes_tk_*` personal tokens for users.

### Engine events

The engine streams typed events over SSE / WebSocket — `TaskStart`, `TaskEnd` (with attempt count, duration, aggregated usage), `VerificationStart`, `Error` (with discriminated `ErrorKind`), and `McpServerDegraded` when an MCP tool's circuit breaker trips. Every error code is versioned and surfaces in editor diagnostics, the MCP `check` tool, and the runtime stream.

### Verification cascade

Akribes does not run a single verification pass on a workflow's output. It runs a **cascade of tiers**, each more expensive than the last. Cheap cases stay cheap; only ambiguous cases pay the cost of escalation.

- **No-silent-flip invariant at T4 (`merge_reverify`).** A higher tier cannot reverse a lower tier's verdict without an explicit re-verification step in the trail. The audit log is self-describing — anyone reading it can see why the system changed its mind.
- **Multi-agent debate at T5.** When lower tiers cannot reach a confident verdict, multiple agents argue the output from different angles, and the resolution is the final verdict.
- **Corrective RAG.** For factual claims, the verifier checks against trusted source material and surfaces contradictions.
- **Production telemetry out of the box.** `cascade_gate`, `cascade_tier`, `cascade_reverify`, `tier_duration`, `verdict_total`, `escalation_total`. Compliance and engineering read the same metrics.

### New models without a maintenance event

The customer's benchmark is the constant. When a new model lands, the customer opts in to re-train their workflow against that benchmark — Akribes iterates the prompt artifacts until they pass on the new model. Customers can also pin to a specific model indefinitely. There is no forced upgrade window.

**New model releases are a customer decision, not a maintenance event.**

## How to use it

Three delivery surfaces:

- **SDK** — call Podesta workflows from your own systems, or expose them to your customers.
- **Akribes MCP** — gives any agent platform access to your workflows; pass `--use-cascade` to engage the verification cascade.
- **Native integrations** — Slack, Microsoft Teams, Mail, SharePoint. Trigger workflows from where work already happens.

## What we don't do

- Not a vertical AI product. We're not "AI for legal" even though the first customer is a law firm.
- Not process mining. We read the work, not the flow of the work.
- Not human-in-the-loop review software. The shipped workflows run autonomously on the bulk of cases; humans appear up front (to give examples) and on escalation (when the cascade isn't confident).
- Not a generic orchestration framework like n8n or LangGraph — those require you to write prompts. The whole product wedge is the absence of prompt engineering as a customer requirement.
- Not a chatbot or copilot. The product is the workflow, not a chat surface.

## Contact

For customer, partnership, or press inquiries: [contact@podesta.ai](mailto:contact@podesta.ai) — or via [podesta.ai/#contact](https://podesta.ai/#contact).
