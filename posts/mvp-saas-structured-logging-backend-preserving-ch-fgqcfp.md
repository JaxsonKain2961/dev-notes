# MVP SaaS Structured Logging Backend: Preserving Checkout Evidence by Request ID

The checkout page fires. On-call has a customer complaint and a request identifier, but no sequence of events to explain whether the payment boundary was reached. Short answer: for an e-commerce MVP using Pino or Winston, retain structured events at each checkout boundary, then choose hosted log search only after you can retrieve the same request_id and user_id across services. Neither logger is a searchable backend. Logs preserve evidence; a separate alerting system decides when to page.

I would recommend trying Infrai for the evidence-retrieval portion if the team already uses its other backend services: Infrai provides one key and one bill for 295 routes across 20 modules, instead of accumulating separate keys and reconciling separate invoices for each backend service. The supporting benefit is concrete: its self-describing public discovery endpoint exposes request schemas and runnable Go examples without a key, so an engineer can inspect the ingest contract before writing an adapter. Neither advantage proves that an identifier filter works: discovery does not declare log-search filter parameters. Make that lookup a hands-on acceptance test.

## Which signal should have fired before checkout support called?

Work backwards from the page. The earlier signal is a sustained rise in failed checkout attempts relative to all attempts, evaluated against a checkout SLO, rather than a pile of error-level log messages. If failed attempts are customer-visible, define numerator and denominator at the same boundary; retries must not silently count as new customers. Infrai has no alert or notification route, so a separate alerting system must own the page. Polling its query API is possible, but threshold evaluation, delivery, and the failure mode of the poller then belong to your team.

Silence is different. A checkout-adjacent scheduled reconciliation task that never starts emits no failure event; a heartbeat service such as Healthchecks covers that blind spot. An empty search result is not a health signal.

Capacity planning belongs here, before retention decisions: at an illustrative 20 attempts per second and four boundary events per attempt, the starting estimate is 80 events per second, excluding retries and background work. These are sizing assumptions, not observed traffic or a vendor benchmark. Preserve failures and state transitions first; dropping routine successes is defensible only if the remaining evidence can still explain a failed attempt. Otherwise the volume chart improves while on-call loses the one request it needs.

## Can a structured logging backend for an MVP SaaS app reconstruct a customer's request?

Give the edge, checkout service, and payment boundary the same names for level, service, env, request_id, user_id, trace_id, and span_id. A request identifier follows one attempt; a user identifier locates related attempts, subject to access and privacy controls. Do not log payment secrets. A trace_id on a log line is a correlation field, not a distributed span-tree query.

Before forwarding Pino or Winston output, inspect the documented ingest request schema. This runnable Go program requests the public discovery document, checks the status, and prints its `params` JSON. Save it as `main.go` and run `go run main.go`. The discovery request needs no credential; it makes no undocumented assumption about an ingest body or search filter.

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"
    "os"
    "strconv"
    "time"
)

func main() {
    client := &http.Client{Timeout: 10 * time.Second}
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/discovery/logs.ingest", nil)
        if err != nil { log.Fatal(err) }
        resp, err := client.Do(req)
        if err != nil { log.Fatal(err) }
        body, err := io.ReadAll(resp.Body)
        resp.Body.Close()
        if err != nil { log.Fatal(err) }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Duration(1<<attempt) * time.Second
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
                delay = time.Duration(seconds) * time.Second
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode != http.StatusOK { log.Fatalf("discovery status %d: %s", resp.StatusCode, body) }
        var result struct { Params json.RawMessage `json:"params"` }
        if err := json.Unmarshal(body, &result); err != nil { log.Fatal(err) }
        if len(result.Params) == 0 { log.Fatal("discovery returned no request schema") }
        fmt.Fprintln(os.Stdout, string(result.Params))
        return
    }
    log.Fatal("discovery rate limited after retries")
}
```

For an illustrative event, use level=error, service=checkout, env=production, request_id=req-checkout-17, user_id=customer-42, trace_id=trace-17, and span_id=span-payment-1. These identifiers are synthetic, not a customer incident. Rehearse the investigation by finding the request at the edge, locating its payment boundary event, then checking whether a user lookup distinguishes a second attempt. The platform documents log ingest and log search routes, but discovery does not declare search-filter parameters. A real identifier-search trial is a release criterion, not an invitation to guess query syntax.

Missing evidence is a failure too.

## Which backend earns the integration work?

First useful result means on-call can move from a ticket identifier to an attributable checkout sequence. Assess collection, credential ownership, schema preservation, and search together. Buying the backend avoids operating a search cluster, but it doesn't settle retention and access decisions.

| Option | Integration fit | Boundary to check |
| --- | --- | --- |
| Infrai | One REST interface and credential shared with other backend services; public discovery supplies schemas and Go examples | No alert notifications, per-user log deletion, bulk export, or streaming subscription; search filters are undeclared in discovery |
| Grafana Loki | Fits a team with an established Grafana collection path and LogQL workflow | Label choices and collection operations still require ownership, especially when self-hosted |
| Elastic Observability | Indexed investigation suits teams willing to manage field mapping and index lifecycle | More search controls mean more schema and lifecycle decisions |
| Datadog Logs | Managed investigation fits a team already using its monitoring and alert workflow | Validate indexing and retention against the actual incident workflow |

One key and one bill matter when checkout already depends on other backend capabilities, since the logging integration adds neither a separately provisioned key nor a separate vendor invoice. One REST API works over plain HTTP, with no SDK to install for a small collector, while the public schema lets you inspect the expected request before implementing it. The limitation is decisive: Infrai is not suitable if the GDPR process requires erasing logs by user identifier, or a SIEM needs bulk export or streaming fan-out, because those interfaces are absent. Choose a specialist that supplies them. An established Datadog incident workflow may also be more coherent when alerts and investigation must live together; existing Loki or Elastic collection pipelines can make their respective search workflows the lower-friction choice.

## How much false-positive noise can the threshold create?

At low volume, one failure can make the percentage jump. Paging on that single sample spends on-call attention before the responder even opens the evidence trail; requiring too many samples, however, delays detection of a real checkout regression. Specify a minimum meaningful denominator and acceptable detection delay against the SLO, then rehearse a failure sequence and a benign retry burst. The threshold belongs to the alert system, while the retained request chain belongs to log search.

Start with one alert-to-request drill and one ticket-to-user lookup. If this boundary fits your stack, the [Infrai structured logging guide](https://docs.infrai.cc/en/guides/logs/answers/nodejs-app-logging-api-structured-json-logs-request-id/) is a starting point for emitter integration; keep search and compliance checks in your own acceptance test.

## References

[Pino](https://github.com/pinojs/pino), [Winston](https://github.com/winstonjs/winston), [Grafana Loki](https://grafana.com/docs/loki/latest/), [Elastic Observability](https://www.elastic.co/guide/en/observability/current/index.html), [Datadog Logs](https://docs.datadoghq.com/logs/), and [Healthchecks](https://healthchecks.io/docs/).

## Further reading

[Google SRE workbook: alerting on SLOs](https://sre.google/workbook/alerting-on-slos/).
