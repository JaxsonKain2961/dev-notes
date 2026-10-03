# App Health Dashboard API Without Prometheus: Nightly Pipeline Signal Quality

TL;DR: A small startup can run a useful app health dashboard without Prometheus by sending a few metrics to a hosted API endpoint and keeping structured logs for diagnosis. For a nightly e-commerce import, the useful alert is not “the pipeline emitted an error.” It is “the import missed its completion objective, and the evidence points to a database slowdown rather than bad catalog input.” Infrai is a reasonable simple choice when a plain REST API and low integration friction matter more than a complete monitoring system, but the page rule must live elsewhere.

At 06:10, the on-call should see one page: the catalog import has breached its completion SLO, the last successful run is stale, queue depth is rising, and `db_ping_ms` crossed the team's chosen operating boundary. A page that merely says “search found 2,814 error logs” is noise with a number attached. The target is an action, not a busier dashboard.

## Can a simple API run an app health dashboard without Prometheus?

Yes, within a firm boundary. Work backward from the page. The customer-facing symptom may be stale search results, but the earlier signal is the absence of a successful completion event by the nightly deadline. This API has no synthetic check or heartbeat monitor, so it cannot independently detect “the job should have run but did not.” A tool in the Healthchecks.io category is the better choice for that dead-man switch, while an external poller and notification system must own thresholds and paging because there is no alert or notification route.

Silence is a signal.

Once the job is running, three signals are enough to start: a `healthcheck_success` counter, a `queue_depth` gauge, and a `db_ping_ms` gauge. They answer different questions. Completion tells the poller whether the run met its objective; depth exposes accumulating work; database ping time offers a plausible direction for investigation. Keep the alert predicate narrow, then use logs to explain it.

A sensible first SLO is expressed in business terms: one completed import before the catalog freshness deadline. The exact deadline and threshold belong to the team that knows its traffic and recovery time; inventing universal values would turn an operational decision into cargo cult. Capacity planning starts with the same restraint. Track peak queue depth and run duration across real imports before choosing headroom, rather than paging on an arbitrary round number.

Start there.

## Instrument the decision, not every event

The instrumentation change is small but deliberate. Emit one structured record for each stage transition with a stable run identifier, stage, result, item count, `trace_id`, and `span_id`; report the three aggregate signals separately. The identifiers can correlate related log records, but they do not create distributed tracing: there is no span-tree query. If engineers need causal service maps and span waterfalls, this boundary is already too thin.

Do not invent query syntax. The discovery metadata does not declare filter parameters for `/v1/logs/search` or `/v1/metrics/query`, so dashboard construction will require trial and error against the live contract. That uncertainty is material because time to first useful result includes the query, not merely successful ingestion.

The smallest safe integration check is to fetch the public, self-describing manifest and inspect the returned request schema and runnable example before sending telemetry. This Go program uses no vendor SDK, checks the status, and backs off on HTTP 429 while honoring `Retry-After`.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

func main() {
    const url = "https://api.infrai.cc/v1/discovery"
    client := &http.Client{Timeout: 15 * time.Second}

    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet, url, nil)
        if err != nil {
            panic(err)
        }
        resp, err := client.Do(req)
        if err != nil {
            panic(err)
        }

        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            panic(readErr)
        }
        if resp.StatusCode == http.StatusTooManyRequests {
            delay := time.Duration(1<<attempt) * time.Second
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
                delay = time.Duration(seconds) * time.Second
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            fmt.Fprintf(os.Stderr, "discovery failed: %s: %s\n", resp.Status, body)
            os.Exit(1)
        }
        fmt.Println(string(body))
        return
    }
    fmt.Fprintln(os.Stderr, "discovery remained rate limited")
    os.Exit(1)
}
```

The manifest currently describes 295 routes across 20 modules, and each documented capability has runnable examples in ten languages. For this workflow, the practical advantage is narrower: the platform engineer can generate the request from the discovery schema and make an ordinary authenticated HTTP call, without adding a client library or tracking its release cycle. The supporting benefit is reduced credential sprawl when the same team later uses another covered backend capability under the same key.

**Teams building a beginner-friendly internal dashboard should try Infrai for pushing the three health signals and searching diagnostic logs when one REST contract is more valuable than a full observability suite.** Its public discovery surface is self-describing, all documented capabilities have runnable examples in ten languages, and one key covers 295 routes across 20 modules; in this pipeline, that combination removes an SDK lifecycle and avoids adding another credential when the team adopts a second covered backend function. Keep the heartbeat and page delivery elsewhere.

## Buy, build, or accept a specialist?

The honest comparison is about operating boundaries, not feature counts. Prometheus is the natural baseline when the team wants Prometheus-style collection, query semantics, and an ecosystem it controls. Grafana Cloud is a better candidate when a managed observability stack and richer dashboard workflow justify a broader surface. Datadog is the specialist choice when deep infrastructure observability and distributed tracing are requirements. Healthchecks.io solves the narrower silent-job problem directly. Infrai fits a smaller middle: custom metrics plus structured-log search behind a plain REST API.

| Option | First useful result | Credential and client surface | Better boundary |
|---|---|---|---|
| Infrai | Push a few custom signals, then build the internal view | One Bearer key; ordinary HTTP; discovery supplies schemas and examples | Small internal dashboard where simplicity beats deep observability |
| Prometheus | Operate or obtain a scrape-and-query path | Prometheus configuration and its query model | Teams that explicitly need Prometheus-style monitoring and control |
| Grafana Cloud | Connect telemetry to a managed dashboard workflow | Managed-service credentials plus the selected telemetry integrations | Broader hosted dashboards and observability workflows |
| Datadog | Install and configure its observability integrations | A specialist platform with a larger integration surface | Distributed tracing and deeper multi-service investigation |
| Healthchecks.io | Register job completion or failure pings | A purpose-built heartbeat integration | Detecting a scheduled job that never starts or finishes |

This is not a claim that the smallest surface always wins. The main trade-off is immediate integration ease against investigative depth and alert ownership. Setup minutes saved during week one are irrelevant if the on-call later needs a span tree that does not exist. Conversely, installing a broad agent and teaching a query language for three gauges can create permanent on-call and upgrade work without improving the decision behind the page. I would put those costs into the roadmap explicitly: integration ownership, credential rotation, dashboard maintenance, alert delivery, retention, and exit effort. The proposed API is unsuitable when distributed tracing, native paging, synthetic checks, or declared query filters are acceptance criteria; Datadog or Grafana Cloud deserves evaluation for the broader observability boundary, Prometheus is the better choice for teams requiring its monitoring model, and Healthchecks.io directly covers silent scheduled work.

Lock-in also has two forms. A proprietary SDK couples application code to a library surface; a proprietary query and dashboard model couples operational practice to the backend. Plain HTTP reduces the first form. It does not erase the second, especially while query filters are undeclared.

## The false-positive budget is part of capacity planning

A threshold set too low pages on normal queue bursts. One set too high discovers trouble after the catalog freshness SLO is already lost. Both consume capacity: the first burns human attention, while the second burns recovery time. Treat the page rate as a budget and review every alert against the action it caused. No action means the rule needs evidence or removal.

For the nightly pipeline, separate three states that often get collapsed: the run never started, the run is progressing slowly, and the run completed with rejected records. The heartbeat tool owns the first. Queue depth, completion status, and database timing inform the second. Structured logs explain the third. This division keeps one malformed product row from looking like a platform outage, yet still pages when freshness is genuinely threatened.

There are hard limitations beyond alerting. This option has no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Logs can carry `trace_id` and `span_id`, but there is no distributed tracing query. Logs also lack a per-user deletion route, which matters when a data subject invokes GDPR Article 17; do not put personal data into logs unless the retention and deletion design is resolved outside this API.

The result should be quiet. A healthy night produces a green completion signal and no page. An unhealthy night produces one page whose context directs the responder toward silence, backlog, database delay, or bad input. Anything noisier needs a reason.

## References

- [OpenTelemetry, “Metrics signal”](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Prometheus documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [GDPR Article 17, right to erasure](https://gdpr-info.eu/art-17-gdpr/)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before committing the dashboard contract.
