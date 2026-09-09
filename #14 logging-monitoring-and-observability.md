# Logging, Monitoring & Observability for Backend Systems

Logging, monitoring, and observability are the practices that let engineers understand what's happening inside a distributed backend system, catch problems before they escalate, and actually pinpoint *why* something went wrong — not just that it did.

## Key Takeaways

- These are **practices implemented on a spectrum**, not a checklist you complete — no system is ever "fully observable," and the right depth depends on team size, resources, and what tools are actually maintained.
- **Observability** rests on three pillars — **logs** (what happened), **metrics** (patterns and trends over time), and **traces** (how a single request moved through different components) — and a system only earns the "observable" label when all three are implemented together, since each answers a different debugging question.
- The practical payoff is a connected workflow: an **alert** on a metric threshold leads you to the related **metrics**, which lead you to the related **logs**, which lead you to the specific **trace** showing exactly where and why a request failed — collapsing what used to be "something is wrong" into "here's exactly what's wrong and where."

## 1. What Is Logging, Monitoring, and Observability?

At a high level, these three closely related practices exist to answer different questions about a running backend system:

- **Logging** is the practice of recording every important event across an application's lifecycle — a journal or diary of what happened, when, and with what context.
- **Monitoring** is having near-real-time (typically delayed by roughly 10–15 seconds, to avoid overwhelming the monitoring system with constant data) visibility into the health and performance of a system — CPU, memory, request throughput, open database connections, and so on.
- **Observability** is the ability to determine the internal state of a system by examining its external outputs. It's a broader, more modern practice built on three pillars: logs, metrics, and traces — a system is only considered observable when all three are in place together.

## 2. Technical Flow (The "Hops")

The video demonstrates how these practices connect into a single debugging workflow, both conceptually and in a real Go-based to-do application instrumented with **New Relic**:

1. An **alert** fires based on a predefined threshold (e.g., error rate exceeding 80%), typically delivered via a webhook to a tool like Slack.
2. The engineer jumps into the **metrics** dashboard to see the concrete numbers behind the alert — request counts, failure counts, average transaction time, throughput.
3. From a specific metric spike (e.g., a jump in errors), the engineer drills into the associated **logs** — the specific log entries tied to that metric, showing the actual error messages and context (user ID, request ID, host, error level, message).
4. From a specific log entry, the engineer jumps into the associated **trace** — the full path a single request took through the system (e.g., middleware → validation layer → service layer → repository → database), pinpointing exactly which component the request failed at.
5. In the video's implementation, a request first hits an **enhanced tracing middleware**, which creates a new "transaction" (New Relic's term for a trace context) and attaches metadata — service name, environment, IP address, user agent, request ID, user ID, tenant ID — before storing that transaction in the request's context.
6. As the request flows into the service layer (e.g., a `CreateTodo` function), the transaction is retrieved from context, additional attributes are added (e.g., the to-do's title, priority), and log statements are emitted at appropriate levels alongside the trace.
7. If the underlying database operation succeeds or fails, that outcome is recorded both as a **log** (with an error or debug/info level) and as an **attribute on the trace**, and a separate structured "business event" log (e.g., "to-do created") captures the outcome with its associated metadata.
8. All of this data flows into the observability platform's dashboard, where logs, metrics, and traces for the same request or transaction are cross-linked, allowing an engineer to move fluidly between "what's the aggregate error rate," "what specific errors occurred," and "what exact code path caused this specific error."

## 3. Why Do We Need Logging, Monitoring, and Observability?

- **Modern backends run in distributed environments** — services spread across multiple servers, regions, and infrastructure components, serving users worldwide, making it impossible to simply "watch" the system directly the way you could with a single local process.
- **Monitoring alone only tells you *that* something is wrong** — traditional, monitoring-only approaches (common roughly a decade ago) can alert you to a problem via thresholds and dashboards, but stop there; they don't tell you the underlying cause.
- **Observability tells you *why*, not just *that*** — with logs, metrics, and traces implemented together, an alert can be traced all the way down to the specific request, component, and failure point that caused it, dramatically shortening debugging time.
- **Faster incident response** — the connected workflow (alert → metric → log → trace) lets engineers move from "something is wrong" to "here's exactly what's wrong and where" without manually correlating disparate data sources.
- **Understanding system behavior over time** — metrics provide historical and current aggregated data (e.g., request volume, failure rates, average latency) that reveal patterns and trends, not just point-in-time snapshots.
- **Collective responsibility** — implementing this properly requires effort both at the application code level (developers instrumenting their code with logs, traces, and metric attributes) and at the infrastructure level (DevOps setting up the collection, storage, and dashboarding systems), making it a shared concern across a team.

## 4. Key Comparisons: Observability vs. Traditional Monitoring-Only Approaches

### Why can't we just rely on monitoring and alerting alone?

1. **Monitoring identifies symptoms, not root causes** — a monitoring-only setup can tell you your error rate crossed a threshold, but gives you no direct path to understanding which specific request, component, or code path is responsible.
2. **It leaves a manual investigation gap** — without connected logs and traces, engineers historically had to manually search through disparate systems (application logs, infrastructure metrics, ad hoc debugging) to reconstruct what actually happened, which is slow and error-prone, especially under incident pressure.
3. **It doesn't scale with distributed systems** — as a backend grows into a system with a handler layer, service layer, validation layer, repository layer, and database layer (potentially across multiple services), monitoring alone can't show how a single request's failure propagated across those components; traces are specifically built to capture that path.
4. **It provides aggregate data without individual-request context** — metrics are excellent for spotting trends (e.g., "error rate is climbing"), but they can't answer "which specific user's request failed, and at exactly which line of code," which is what logs and traces are for.
5. **The gap is closed, not eliminated, by observability** — it's worth noting that observability doesn't replace monitoring; it extends it. Metrics (closely tied to what "monitoring" traditionally measured) remain one of the three pillars — the addition is connecting them to logs and traces so the *why* is available alongside the *what*.

## 5. Deep Dive: Logging Practices, the Three Pillars, and a Real Implementation

### 5.1 Log levels

Most logging libraries support assigning a **severity level** to each log entry, and the common levels described are:

- **Debug** — highly detailed, used only in development for active troubleshooting; typically disabled in production because of the volume and noise it would generate.
- **Info** — general application operations and successful business events (e.g., "a to-do was created").
- **Warn** — events that are neither fully successful nor critical errors — the video's example is a failed login attempt due to a wrong password, which is expected user behavior rather than a system fault.
- **Error** — genuine failures: validation errors, failed database queries, and similar issues — one of the primary reasons logging exists in the first place.
- **Fatal** — the most severe level, indicating a serious issue that typically causes the application to stop (and potentially restart, depending on infrastructure configuration).

### 5.2 Structured vs. unstructured logging

- **Unstructured (console) logs** — human-readable, often colorized plain text, used in local development because it's easier to visually scan and spot issues while actively coding.
- **Structured logs** — most commonly formatted as **JSON**, used in production because log management and aggregation tools (e.g., the ELK stack, or the Grafana/Loki/Promtail stack) need to reliably parse fields like user ID, request ID, and error codes out of each entry; plain text is difficult and error-prone for these tools to parse at scale, while JSON's key-value structure makes field extraction straightforward.

The video's demo application implements this as a simple environment-driven configuration switch — a `local` (console/plain-text) mode and a `production` (JSON) mode for the same logger, illustrating this trade-off directly rather than just describing it.

### 5.3 The three pillars in practice

- **Logs** — the recorded events themselves (what happened, when, with what context).
- **Metrics** — quantifiable numbers tracked over time and in near-real-time: request counts, failure counts (commonly defined as any response with a status code outside the 2xx range), average transaction time, throughput, and custom business-specific counts (e.g., how many to-dos were created versus how many creation attempts failed).
- **Traces** — the record of a single request's journey through a system's components (e.g., middleware → validation → service → repository → database), showing where in that journey a failure occurred. This is the pillar most directly tied to **observability** as a distinct concept from basic monitoring.

### 5.4 Instrumentation and OpenTelemetry

- **Instrumentation** is the practice of actually adding the code that measures and records these attributes (creating transactions, adding attributes, emitting logs) within an application — it's the mechanism by which a system becomes observable in practice, not just in theory.
- **OpenTelemetry (OTel)** is described as a recent, open standard providing a broad ecosystem of tools, SDKs, and best practices for instrumenting applications consistently across languages (Go, Node.js, Python, etc.), independent of which backend/vendor ultimately receives the data. This description holds up well against current status: OpenTelemetry has become the dominant, CNCF-backed, vendor-neutral standard for telemetry collection, with traces, metrics, and logs now stable across all major language SDKs, and even major proprietary vendors (including New Relic itself, and Datadog) now accept OpenTelemetry-formatted data as a standard input — reinforcing the video's point that even when using a proprietary tool, integrating an OpenTelemetry collector is possible and increasingly the norm for retaining more control over instrumentation. As a currency note beyond the video's original 2024-era framing: OpenTelemetry has since added a fourth signal, **continuous profiling**, alongside logs, metrics, and traces, though this wasn't part of the original three-pillars framing the video describes.

### 5.5 Tooling landscape

Two broad approaches were presented:

- **Open-source stack** — Grafana (dashboard/visualization layer), Prometheus (metrics backend), Loki and Promtail (log aggregation), and Jaeger (distributed tracing). This combination is commonly referred to as "the Grafana stack" and remains a standard, actively maintained open-source observability toolchain.
- **Proprietary all-in-one platforms** — services like **New Relic** or **Datadog**, which bundle logging, monitoring, and tracing into a single managed product. The video frames this as the more practical choice for teams without the size or dedicated resources to configure and maintain a full open-source stack themselves — a reasonable, still-current trade-off: self-hosted observability stacks carry real operational overhead, and managed platforms trade cost for reduced maintenance burden.

### 5.6 Worked example: a New Relic–instrumented Go to-do application

The video walks through a real Go backend with New Relic integration to ground these concepts:

- A **logger initialization file** configures the log level based on environment (`info` in production, `debug` in local development) and the log format (`console`/text locally, JSON in production) — directly demonstrating the level and structured-vs-unstructured concepts described above.
- A **New Relic middleware**, applied at the router level, instruments every incoming request — creating a new transaction and attaching metadata such as service name, environment, IP address, user agent, request ID, user ID, tenant ID, and user email, then storing that transaction in the request's context so downstream layers (like the service layer) can retrieve and extend it.
- Within a `CreateTodo` service function, the transaction is pulled from context, additional attributes (the to-do's title, and priority if provided) are attached to it, and info/debug-level log statements are emitted at each meaningful step (starting the operation, validating parent/child relationships, executing the database write). On failure, an error-level log is emitted and the error is attached to the transaction; on success, the created to-do's ID is logged and attached, plus a separate structured business-event log capturing the full outcome (ID, title, category ID, priority).
- In the New Relic dashboard, triggering an unauthorized request (missing auth token) against the API produces a visible **error metric** (e.g., roughly 80% error rate shown in the errors view), which links directly to the specific **log entry** for that failed request (showing application name, environment, error code, host, IP, level, message, HTTP method, route, and timestamp), which in turn links to the full **trace** for that request and the underlying **transaction** view (showing error rate and response time specifically for that route), and further down to **runtime metrics** for the Go application itself (garbage collection time, memory usage, throughput, average response time).

This walkthrough demonstrates, concretely, the alert → metric → log → trace workflow described conceptually earlier: a single dashboard interaction chain that takes an engineer from "something's wrong" to "here's the exact request, component, and cause."

---

## 6. Interview Questions & Key Concepts

### Fundamentals

**Q: What's the difference between logging, monitoring, and observability?**
*Logging is recording individual events with context (what happened, when, to whom). Monitoring is near-real-time visibility into system health and performance metrics (CPU, memory, throughput). Observability is the broader capability — built on logs, metrics, and traces together — to determine a system's internal state and root cause of a problem purely from its external outputs, rather than just being alerted that a problem exists.*

**Q: What are the three (or four) pillars of observability, and what distinct question does each answer?**
*Logs answer "what happened" with detailed event context. Metrics answer "what's the pattern or trend" with quantified, aggregated numbers over time. Traces answer "how did this specific request move through the system, and where did it fail" by capturing the full path across components. As of 2026, OpenTelemetry has added a fourth signal — continuous profiling — capturing fine-grained resource/CPU usage within running code, though the classic "three pillars" framing (logs, metrics, traces) is still the foundational mental model most engineers start from.*

**Q: What are common logging levels, and how should a team decide what level to log at in production versus development?**
*Common levels, in increasing severity, are debug, info, warn, error, and fatal. Debug is typically only enabled in development due to its verbosity; production systems commonly default to info level and above, so routine successful operations are still visible but noisy debug detail doesn't clutter production logs or drive up log storage costs.*

### Architecture & Security

**Q: Why is structured (e.g., JSON) logging preferred in production over plain-text console logs?**
*Log aggregation and management tools (ELK, Grafana/Loki, Datadog, New Relic, etc.) need to reliably extract specific fields — user ID, request ID, error codes — from every log line to power search, dashboards, and alerting. Plain text is easy for humans to scan in a local terminal but difficult and error-prone for tools to parse reliably at scale, whereas JSON's key-value structure makes field extraction deterministic.*

**Q: What is OpenTelemetry, and why has it become significant for observability architecture?**
*OpenTelemetry is an open-source, vendor-neutral standard (a CNCF project) for instrumenting applications to produce logs, metrics, and traces in a consistent format, regardless of the language used or which backend ultimately receives the data. Its significance has grown substantially since this video was made: it's now the de facto default for cloud-native instrumentation in 2026, with traces, metrics, and logs stable across all major language SDKs, and even proprietary vendors like New Relic and Datadog now accept OpenTelemetry-formatted data — meaning teams can instrument once and switch or combine backends without rewriting instrumentation code, reducing long-term vendor lock-in.*

**Q: What security or data-handling concerns should be considered when implementing logging and tracing?**
*Logs and traces often capture rich contextual metadata (user IDs, IP addresses, request payloads), which can inadvertently include sensitive data (passwords, tokens, personal information) if instrumentation isn't deliberate about what it captures. Current best practice — consistent with general secure-logging guidance beyond what this video covers in depth — is to avoid logging sensitive fields directly, apply redaction/sanitization at the instrumentation layer, and ensure access to logging/observability dashboards themselves is access-controlled, since these systems become a rich, centralized source of user and system data.*

### Business Logic & Operational Practices

**Q: How would you design an alert-to-resolution workflow using logs, metrics, and traces together?**
*Configure an alert on a meaningful metric threshold (e.g., error rate exceeding a defined percentage) delivered to a notification channel; from that alert, drill into the metrics dashboard to see the scope and pattern of the issue; from a specific metric spike, jump to the associated log entries to see actual error details and context; and from a specific log entry, follow its linked trace to see exactly which component or function in the request's path caused the failure. This connected chain is the core practical value of implementing observability rather than monitoring alone.*

**Q: Why is implementing logging, monitoring, and observability described as a "spectrum" rather than a completed checklist?**
*No system is ever fully or completely observable — teams implement these practices to varying depth based on team size, available resources, and which tools they can realistically maintain, and the "right" level of investment differs across organizations and even across services within the same organization. This framing matters practically: it means engineers shouldn't feel they've "failed" at observability for not having every metric, trace, and log implemented everywhere — the goal is meaningful coverage of what actually helps debug real incidents, not exhaustive instrumentation of everything.*

**Q: When would a team choose a proprietary all-in-one observability platform (e.g., New Relic, Datadog) over an open-source stack (e.g., Grafana, Prometheus, Loki, Jaeger)?**
*Proprietary platforms reduce operational overhead — no need to deploy, configure, and maintain multiple separate open-source components — which matters most for smaller teams or teams without dedicated infrastructure/DevOps capacity. Open-source stacks offer more control and no per-seat/usage-based vendor cost at scale, which can matter more for larger teams with the resources to operate them. Since both approaches increasingly speak OpenTelemetry natively, teams aren't as locked into this choice long-term as they once were — a system can start on one and migrate data collection more easily than in the past.*

---

*Approximate word count: 3,050 words.*
