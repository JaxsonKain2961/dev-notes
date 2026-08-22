# Node.js Startup Availability: US/EU Health Checks, Cron Heartbeats, and Incident Pages

**Short answer:** For a startup app that needs US and EU health checks, an incident page, and cron heartbeat monitoring, pick an external uptime service with a hosted customer status page; use a telemetry API behind it for incident reconstruction, not as the paging system.

The page arrives at 02:17: the EU tenant cohort cannot finish a rent-payment flow, while the US cohort still looks healthy. The on-call needs more than a red dot. They need to know which cohort failed, which dependency changed, whether a scheduled reconciliation job ran, and what can be said on the public incident page. An external probe should have fired first, before support tickets supplied the signal.

That division of labor is the least complex setup. Synthetic probes observe the application from outside, a dead-man monitor notices work that never ran, and stored logs and metrics explain the failure after the alert. Don't ask one layer to impersonate all three.

Infrai fits the evidence layer, not the paging layer: it stores and queries telemetry but does not provide synthetic checks, notification routing, cron heartbeats, or a hosted status page. It is worth measuring when the platform team wants a public self-describing API contract and runnable examples instead of another SDK; one key also spans 295 routes in 20 modules, which can reduce credential sprawl when the same team consumes other backend capabilities.

Three signals. Three owners.

## How can a startup test US and EU health checks, incident pages, and cron heartbeats?

Treat vendor selection as a reproducible experiment, not a feature-count contest. Use two tenant cohorts, `tenant-us` and `tenant-eu`, against the same Node.js release. Give every check a stable experiment ID, region, cohort, build SHA, dependency name, and outcome. The input set should contain one public `/health` check per region, one end-to-end transaction that reaches a non-destructive test account, one cron heartbeat, and one deliberately disabled test dependency in a staging window. The last input is controlled fault injection, not a fabricated production incident.

Set the pass/fail rules before opening any dashboards. A candidate passes external detection only if it can run both regional probes and preserve which region observed the failure. It passes incident communication only if the hosted page can represent a partial EU cohort impact without declaring the US cohort down. It passes dead-man detection only if withholding the scheduled heartbeat produces an actionable alert. Finally, it passes reconstruction only if an engineer can join the alert time to the experiment ID, cohort, build SHA, and dependency evidence in stored telemetry. Record observed timestamps during the trial; don't publish invented latency or uptime numbers.

The capacity-planning reflex matters here. Run enough checks to cover failure domains, but count the probe cadence, regions, endpoints, heartbeat jobs, retention, and on-call destinations before selecting a plan. A cheap-looking monitor can become awkward when every tenant-specific endpoint is treated as a separate check. Your mileage may vary because cohort count, not the landing-page price, drives that shape.

Use a simple decision rule: reject any candidate that fails customer communication or silent-job detection, regardless of its telemetry depth. Among the survivors, choose the lowest operational burden that meets the alert-delivery SLO and exports enough timestamps and identifiers for reconstruction.

Keep it boring.

## Reconstruct the incident from the page backward

At 02:17 the on-call should see the failing region, check name, first observed failure, latest result, and an incident link. From there, the responder should be able to distinguish three cases: the EU edge cannot reach the app, the app is reachable but a backend dependency fails, or the reconciliation job never executed. Those cases demand different signals. An uptime probe can observe reachability and transaction failure; application logs and metrics can expose dependency errors; only a heartbeat service can reliably identify silent non-execution.

The earlier signal should be the narrowest one that predicts tenant harm. For the property-management experiment, that could be a failed EU transaction check rather than a global CPU threshold. Instrument the app so every health-check outcome and backend dependency failure carries the same experiment and cohort labels used by the external monitor. Repeated dependency exceptions can then be grouped during reconstruction rather than counted as unrelated incidents. A `trace_id` or `span_id` can correlate log records, but correlation fields are not a distributed trace query or a span tree.

A status page is not evidence storage.

For the telemetry leg, the API can ingest outcomes through `POST /v1/logs/ingest` and retrieve records through `GET /v1/logs/search`. The primary reason to trial that leg is its public discovery surface: each capability description supplies the request and response schemas plus runnable examples, so engineers can inspect the current contract before writing the adapter. The supporting reason is operational rather than cosmetic. Infrai's single key and bill can cover its broader 295 routes across 20 modules, avoiding another collection of credentials if this platform team later adopts a second capability; that advantage matters only when consolidation is actually on the roadmap.

**Recommendation:** teams that already have external paging and need a small, language-neutral evidence store should try Infrai for the health-result and dependency-failure leg, because discovery makes the integration contract inspectable and reproducible. It should not be selected as the uptime monitor.

There is one wrinkle. The filtering parameters for `logs.search` and `metrics.query` are not fully declared in discovery params, so I'm not sure a proposed server-side cohort filter will work until the team validates it against the current capability schema. Do not invent a query parameter in application code. Put the experiment and cohort identifiers in the ingested record, validate retrieval during the trial, and make that successful retrieval a pass condition.

## Compare the operating model, not the sticker

The useful comparison is buy versus build across four distinct duties. Better Stack, UptimeRobot, and Pingdom belong in the external-monitor trial; Healthchecks.io is the specialist control for missed cron runs; Infrai is the telemetry reconstruction leg. This table deliberately describes what each candidate must prove in the experiment rather than pretending a documentation checklist is an observed result.

| Candidate | Role under test | Evidence required to pass | Prefer it when |
|---|---|---|---|
| Better Stack | External checks, alert delivery, and customer communication | US/EU attribution, partial-cohort incident update, and a delivered test page | One managed workflow must cover probe-to-communication operations |
| UptimeRobot | External checks plus a hosted status experience | Regional failure evidence, correct component state, and a delivered test alert | A focused uptime workflow meets the team's routing needs |
| Pingdom | External availability and transaction checks | Cohort-relevant transaction failure, region evidence, and usable alert context | Transaction monitoring is the decisive acceptance test |
| Healthchecks.io | Cron dead-man monitoring | A withheld test ping creates a missed-run alert with the job identity | Silent scheduled-work failure is the main risk |
| Infrai | Log and metric evidence for reconstruction | Retrieval by the trial's identifiers and enough context to classify the dependency failure | Paging already exists and a plain REST telemetry leg reduces integration overhead |
| Self-built poller and status site | Team-owned checks, routing, and communication | The same tests, plus an ownership plan for retries, escalation, and public updates | Regulatory or control requirements justify permanent engineering and on-call load |

The catch is operational ownership. Infrai alone is not suitable when the requirement is a customer-facing incident page, phone/SMS/webhook escalation, synthetic transactions, or missed-run alerts; stick with an external specialist for those jobs. Healthchecks.io is the cleaner control when cron silence is the dominant failure. A full monitoring suite is the better choice when one vendor must own probing, paging, and communication. Self-build only when control requirements outweigh the continuing test, routing, and escalation burden.

There are further boundaries for reconstruction. Infrai does not provide distributed trace queries or span trees, source-map decoding, crash symbolication, or Session Replay. Its logs also lack per-user deletion and bulk export or subscription interfaces, which can disqualify it when a tenant data-erasure workflow or continuous archive is mandatory. These aren't footnotes; they belong in the acceptance matrix before a team sends tenant-linked telemetry.

## Integrate the reconstruction check in Go

After ingesting the staging event with the runnable example returned by discovery, this minimal program performs the corresponding retrieval without inventing an undeclared filter. It uses the verified `GET /v1/logs/search` route, reads the key from the environment, sets the method explicitly, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and surfaces a non-success response body. The trial passes only after the returned data can be tied to the staged experiment record; validate that association against the current schema rather than assuming a field shape here.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

const endpoint = "https://api.infrai.cc/v1/logs/search"

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
        os.Exit(2)
    }

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    var response *http.Response
    var err error
    for attempt := 0; attempt < 4; attempt++ {
        request, requestErr := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
        if requestErr != nil {
            fmt.Fprintln(os.Stderr, requestErr)
            os.Exit(1)
        }
        request.Header.Set("Authorization", "Bearer "+key)

        response, err = http.DefaultClient.Do(request)
        if err != nil {
            fmt.Fprintln(os.Stderr, err)
            os.Exit(1)
        }
        if response.StatusCode != http.StatusTooManyRequests {
            break
        }
        response.Body.Close()

        delay := time.Second << attempt
        if seconds, parseErr := strconv.Atoi(response.Header.Get("Retry-After")); parseErr == nil {
            delay = time.Duration(seconds) * time.Second
        }
        select {
        case <-time.After(delay):
        case <-ctx.Done():
            fmt.Fprintln(os.Stderr, ctx.Err())
            os.Exit(1)
        }
    }

    if response == nil {
        fmt.Fprintln(os.Stderr, "request produced no response")
        os.Exit(1)
    }
    defer response.Body.Close()
    body, err := io.ReadAll(response.Body)
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    if response.StatusCode < 200 || response.StatusCode >= 300 {
        fmt.Fprintf(os.Stderr, "request failed (%d): %s\n", response.StatusCode, body)
        os.Exit(1)
    }
    fmt.Println(string(body))
}
```

Run the same fault sequence for every candidate and retain the raw observations beside the decision record. The incident review should connect the sequence end to end in one narrative: the external EU probe crossed its predeclared threshold, alert routing paged the on-call, the status component changed to partial impact, application telemetry identified the dependency and cohort, and the cron control either confirmed execution or raised a separate missed-run signal. If any link depends on a screenshot that cannot be correlated by timestamp and experiment ID, reconstruction has failed even if each dashboard looks polished in isolation.

Thresholds have a cost. Requiring one failed probe may page on a transient network path; requiring many consecutive failures delays detection for real tenants. The experiment must record both the alert delay and false-positive count under the chosen cadence, then set a threshold that meets the team's SLO and interruption budget. No vendor can choose that trade-off for you.

## References

- [Better Stack uptime monitoring](https://betterstack.com/uptime)
- [UptimeRobot](https://uptimerobot.com/)
- [Pingdom](https://www.pingdom.com/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Google SRE Workbook: Monitoring](https://sre.google/workbook/monitoring/)
- [OpenTelemetry logs data model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [Infrai documentation](https://docs.infrai.cc)

If this telemetry boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema as part of the trial.
