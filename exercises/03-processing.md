# Exercise 03 — Processing telemetry

[← Exercise 02](02-setup-obi.md)

Filter noisy spans in the collector, use an OTTL transform processor to anonymize sensitive span attributes, and swap head-based sampling for tail-based sampling that keeps every error and every slow trace.

## Contents

- [What you will change](#what-you-will-change)
- [Part 1 — Filter frontend noise in the collector](#part-1--filter-frontend-noise-in-the-collector)
  - [Step 1.1 — Add a filter processor](#step-11--add-a-filter-processor)
  - [Step 1.2 — Wire the processor into the traces pipeline](#step-12--wire-the-processor-into-the-traces-pipeline)
- [Part 2 — Tail-based sampling in the collector](#part-2--tail-based-sampling-in-the-collector)
  - [Step 2.1 — Add a tail-sampling processor](#step-21--add-a-tail-sampling-processor)
  - [Step 2.2 — Wire the processor into the traces pipeline](#step-22--wire-the-processor-into-the-traces-pipeline)
- [Verify](#verify)
- [Catch up](#catch-up)

## What you will change

| File                                                                                                                                                | Changes                                                       |
| --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| [otel-collector/config.yaml](https://github.com/mariomac/devopsdays-bcn-2026-otel/blob/03-processing/otel-collector/config.yaml) | Drop static-file and `/health` spans from the frontend        |
| [otel-collector/config.yaml](https://github.com/mariomac/devopsdays-bcn-2026-otel/blob/03-processing/otel-collector/config.yaml) | Keep only error and slow traces via `tail_sampling`           |
| [docker-compose.yaml](https://github.com/mariomac/devopsdays-bcn-2026-otel/blob/03-processing/docker-compose.yaml)               | Remove the frontend's head sampler so the collector sees 100% |

---

## Part 1 — Filter frontend noise in the collector

[Exercise 02](02-setup-obi.md) instrumented all the signals of the selected services,
without any kind of filtering.

To remove noise and lower the telemetry costs, we are going to use the OpenTelemetry
Collector's [filter processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/v0.149.0/processor/filterprocessor), which uses [OTTL](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/v0.149.0/pkg/ottl) expressions to drop some traces that might be noisy: health checks and downloads of assets (icons, css, images...).

> [!WARNING]
> **Only drop leaf spans.** The collector cannot rewrite `parent_span_id` references. Dropping a parent orphans its children, breaking the trace tree. Static file requests and health-check pings are safe targets; for anything else prefer disabling the module or an instrumentation filter in the application.

### Step 1.1 — Add a filter processor

In [otel-collector/config.yaml](https://github.com/mariomac/devopsdays-bcn-2026-otel/blob/03-processing/otel-collector/config.yaml):

```diff
# otel-collector/config.yaml
+  filter/drop_health_check:
+    error_mode: ignore
+    traces:
+      span:
+        - >-
+          (attributes["url.path"] == "/health" or
+          IsMatch(attributes["url.path"], "\\.(css|js|ico|png|jpg|jpeg|gif|svg|woff|woff2|ttf|eot|map)(\\?.*)?$"))
```

The condition drops a span when all three are true: service is `frontend`, span kind is `SERVER`, and `url.path` is `/health` or a static file extension.

### Step 1.2 — Wire the processor into the traces pipeline

```diff
# otel-collector/config.yaml
     traces:
       receivers: [otlp]
-      processors: [resourcedetection, batch]
+      processors: [resourcedetection, filter/drop_health_check, batch]
       exporters: [otlp_http]
```

The processor runs before `batch`, so filtered spans never reach the exporter.

---

## Part 2 — Tail-based sampling in the collector

Sampling allows lowering your telemetry costs by dropping a percentage of the
traces, so you don't have to submit and store each invocation to your service.

You could configure OBI (or any other OTEL SDK) to implement **head-based sampling**:
the SDK decides to drop a percentage of traces before they are created and submitted.
It's cheap but uninformed: the decision is made before any error or latency signal exists,
so a fixed fraction of incidents is dropped unconditionally.

The [tail sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/v0.149.0/processor/tailsamplingprocessor) defers the decision until the trace is observable. The collector buffers every span of a trace in memory and, after a `decision_wait` window elapses, evaluates policies (e.g. status code, latency) against the trace and exports the ones that match.

### Step 2.1 — Add a tail-sampling processor

In [otel-collector/config.yaml](https://github.com/mariomac/devopsdays-bcn-2026-otel/blob/03-processing/otel-collector/config.yaml):

```diff
# otel-collector/config.yaml
+  tail_sampling:
+    decision_wait: 10s
+    num_traces: 50000
+    expected_new_traces_per_sec: 100
+    policies:
+      - name: errors
+        type: status_code
+        status_code:
+          status_codes: [ERROR]
+      - name: slow_traces
+        type: latency
+        latency:
+          threshold_ms: 1000
```

Top-level policies are OR-ed: a trace is kept if **any** policy matches. With the two above, every error trace and every trace over 1 s is kept; everything else is dropped outright — no probabilistic fallback.

- `decision_wait: 10s` — how long to buffer before deciding. Must exceed your slowest expected trace, or late spans arrive after the decision and get dropped.
- `num_traces: 50000`, `expected_new_traces_per_sec: 100` — buffer sizing. Tail sampling trades RAM for smarter decisions; tune these for your workload.

### Step 2.2 — Wire the processor into the traces pipeline

```diff
# otel-collector/config.yaml
     traces:
       receivers: [otlp]
-      processors: [resourcedetection, filter/drop_health_check, batch]
+      processors: [resourcedetection, filter/drop_health_check, tail_sampling, batch]
       exporters: [otlp_http]
```

Order matters: `filter` runs first (cheap drop — no point buffering spans we'll discard anyway), then `tail_sampling` makes the keep/drop call on the rest, and `batch` runs after sampling.

---

## Verify

```bash
docker compose up --build
```

Make sure `CHAOS_MODE=true` is set in your `.env` (it's the default) — chaos mode produces the error and slow traces that tail sampling is meant to keep. Log in as `alice` and exercise the app (restaurant list, detail pages, search). Wait ~15 s after your last click so the `decision_wait` window closes and buffered traces can be emitted.

**Part 1** — static-file and frontend `/health` spans should be gone:

```traceql
{ resource.service.name = "frontend" && kind = server && span.url.path =~ ".*(css|js|ico|png|health)" }
```

**Part 2** — Tempo should now contain **only** error traces and slow traces, with nothing boring in between.

_Signal is present_ — errors and slow traces survived sampling — [Open in Grafana (Tempo)](http://localhost:3000/explore?schemaVersion=1&orgId=1&panes=%7B%22abc%22%3A%7B%22datasource%22%3A%22tempo%22%2C%22queries%22%3A%5B%7B%22refId%22%3A%22A%22%2C%22queryType%22%3A%22traceql%22%2C%22query%22%3A%22%7B%20status%20%3D%20error%20%7D%20%7C%7C%20%7B%20duration%20%3E%201s%20%7D%22%7D%5D%2C%22range%22%3A%7B%22from%22%3A%22now-1h%22%2C%22to%22%3A%22now%22%7D%7D%7D):

```traceql
{ status = error } || { duration > 1s }
```

_Noise is gone_ — the interesting one — [Open in Grafana (Tempo)](http://localhost:3000/explore?schemaVersion=1&orgId=1&panes=%7B%22abc%22%3A%7B%22datasource%22%3A%22tempo%22%2C%22queries%22%3A%5B%7B%22refId%22%3A%22A%22%2C%22queryType%22%3A%22traceql%22%2C%22query%22%3A%22%7B%20kind%20%3D%20server%20%26%26%20status%20%3D%20ok%20%26%26%20duration%20%3C%20100ms%20%7D%22%7D%5D%2C%22range%22%3A%7B%22from%22%3A%22now-1h%22%2C%22to%22%3A%22now%22%7D%7D%7D):

```traceql
{ kind = server && status = ok && duration < 100ms }
```

Fast successful server spans should **not** appear. A handful of hits are acceptable if the query window crosses the pre-sampling period, or if a fast span belongs to a trace that tail sampling kept for a different reason (a sibling span exceeded the latency threshold — the whole trace is kept together).

Check out the [metrics drilldown](http://localhost:3000/a/grafana-metricsdrilldown-app/), [traces drilldown](http://localhost:3000/a/grafana-exploretraces-app/), and [logs drilldown](http://localhost:3000/a/grafana-lokiexplore-app/) — great tools to see what telemetry is available.

---

## Learn more

- [Transforming telemetry in the Collector](https://opentelemetry.io/docs/collector/transforming-telemetry/) — guide to the transform processor and OTTL
- [OTTL Playground](https://ottl.run/) — experiment with OTTL expressions interactively
- [Tail Sampling Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/v0.149.0/processor/tailsamplingprocessor) — every policy type and tuning knob
- [OTel sampling](https://opentelemetry.io/docs/concepts/sampling/) — head vs tail, tradeoffs, and when to use each
- [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib) — source of the `filter`, `transform`, and `tail_sampling` processors

---

## Catch up

```bash
git checkout origin/03-processing
```

---

[← Exercise 02](02-setup-obi.md)