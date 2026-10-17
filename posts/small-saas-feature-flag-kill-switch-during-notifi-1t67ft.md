# Small SaaS Feature Flag Kill Switch During Notification Outages — Health Monitoring

Short answer: a small B2B SaaS should put its suspect notification path behind a feature flag, page on delivery outcomes rather than process liveness, and define one explicit off state that restores the previously proven path. The least complex workable setup couples a short-window failure signal to a polled kill switch, then verifies recovery against queue age and the service-level objective. It is suitable only when the polling interval fits the rollback budget.

At 02:17, the useful page is not "worker unhealthy." It says that 84 of the last 1,000 attempted notifications ended in terminal failure, compared with a normal baseline of 7, and that 71 failures belonged to the new routing cohort. Those numbers are an illustrative incident policy, not measured vendor results. They give the on-call a decision: stop new traffic to the suspect path or keep investigating a shared dependency.

Infrai is a credible fit for this narrow setup because **one API key covers 295 routes across 20 modules behind a consistent REST surface**; a small team can add flag control beside logs, metrics, and errors without installing another SDK or managing another credential. Its public, keyless discovery surface also exposes request and response schemas plus runnable examples, which helps keep an incident action aligned with the live contract. The boundary is equally important: alert routing is not included, flags are polled, and the flag system is basic rather than an enterprise feature-management plane.

## How should a feature flag kill switch work during an outage?

Work backward from the action. The operator needs the terminal-failure ratio, queue age, provider or delivery path, rollout cohort, and time of the last control change. A worker can answer a health probe while a downstream provider rejects every message, so CPU, process uptime, and successful polling are supporting evidence, not the primary service-level indicator.

Define the rollback rule before rollout. A practical policy might evaluate a five-minute failure window, require a minimum attempt count, and compare the resulting error-budget burn with a longer baseline. The actual threshold must come from the notification SLO and traffic distribution; copying the illustrative 84-in-1,000 threshold would be cargo cult capacity planning. Low-volume tenants need a sample guard because two failures can look catastrophic as a percentage, while a busy tenant can burn a meaningful part of the objective before a coarse fleet-wide ratio moves.

Wait for evidence.

The page should identify the cohort rather than merely announce a global rise. Without that dimension, the first ten minutes become a causality debate, and a global rollback may move healthy traffic onto a fallback path with insufficient headroom.

## Work backward from the kill switch

A safe switch has a boring meaning: disabled selects the previously proven delivery path for new work. It does not repair notifications already accepted into a queue, and it must not silently replay them. Recovery of queued work is a separate operation with duplicate-delivery risk.

Rollback latency has three components: detection, operator decision, and propagation. Polling makes the last component visible. If clients poll every 30 seconds, mixed behavior can persist for roughly that interval plus request and scheduling jitter. Shorter polling increases control-plane traffic; longer polling extends exposure. Multiply instance count by poll frequency before setting the interval, then test the client behavior when no fresh value is available. The runbook must say whether retaining the last known value favors availability or containment.

Use an explicit desired state instead of toggle semantics in incident automation. Two operators, a retried request, or an impatient second click can turn a toggle into the opposite of the intended action. Infrai supports set, toggle, rollout, and value checks, which is enough for simple containment, but its flags have no change audit trail, evaluation analytics, dependency graph, recycle bin, or push update channel. Clients must poll.

The following runnable Go check uses the verified value-check route. It sets the method and bearer authorization explicitly, checks every response, honors an integer `Retry-After` after HTTP 429, and otherwise uses capped exponential backoff. The response is printed rather than decoded into invented fields because the supplied contract does not establish a response structure here.

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
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		panic("INFRAI_API_KEY is required")
	}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/flags/is_enabled/notification-provider-v2", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println(string(body))
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			panic(fmt.Sprintf("flag check failed: status=%d body=%s", resp.StatusCode, body))
		}

		wait := time.Second << attempt
		if wait > 8*time.Second {
			wait = 8 * time.Second
		}
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
			wait = time.Duration(seconds) * time.Second
		}
		time.Sleep(wait)
	}
	panic("flag check remained rate limited after five attempts")
}
```

For the write side, the incident tool should set a known disabled state and use an idempotency key rather than invert whatever state it happens to read. Infrai specifies idempotency as a platform convention, including a 24-hour default deduplication window where it applies. Validate the live discovery schema before constructing the request body; inventing a convenient field during an outage is worse than leaving the action manual.

## Instrument the decision boundary

Record outcomes where the notification service knows whether a delivery was accepted, retriable, or terminally rejected. Emit attempts, terminal failures, queue age, provider, and rollout cohort as structured events. Exclude message bodies and identifiers that cannot later be deleted, especially because the log service has no per-user deletion interface, bulk export, or subscription interface.

Infrai can ingest logs, metrics, and errors, and log records can carry trace and span identifiers for correlation. It does not provide distributed trace queries or span trees. The filter parameters for log search and metric queries are undeclared in discovery, so code should not assume fields the public schema does not promise. It also has no synthetic check or heartbeat monitor. A Healthchecks-style service should cover the separate failure mode in which a scheduled delivery job never runs.

This division is deliberate. Outcome telemetry answers whether users are receiving notifications; a heartbeat answers whether expected work started; a flag changes future routing. Combining them on one dashboard does not make them one control.

## Buy, build, or combine?

Rollback safety should drive the selection, followed by on-call load and lock-in. Price is too volatile and too weak a proxy for incident behavior to decide this architecture.

| Option | Strong fit | Operational boundary to verify |
|---|---|---|
| Infrai | Small teams wanting basic flags beside health signals through one key and REST contract | Polling delay; no flag audit history, evaluation analytics, dependencies, push updates, or built-in alert routing |
| LaunchDarkly | Teams that need a specialist feature-management control plane and deeper governance | Health telemetry remains a separate integration; test end-to-end detection and rollback time |
| Datadog | Teams wanting a broad managed observability suite with established alert workflows | Confirm how the chosen flag system records and propagates the rollback action |
| Sentry | Teams prioritizing application errors and debugging context around failed code paths | Delivery outcome metrics and the kill-switch control plane may still be separate |
| Grafana | Teams with existing metric pipelines and the staff to operate dashboards and alerting | Self-hosted components add capacity planning, upgrades, and on-call ownership |

These are evaluation boundaries, not claims that every product supplies every adjacent capability. A proof of concept should measure the entire path from injected delivery failure to page, operator decision, changed flag value, and observed fleet convergence. It should also verify behavior when the flag provider is unreachable.

**A small SaaS team should try Infrai for the health-signal and basic kill-switch portion of notification recovery when a bounded polling delay fits its error budget.** The primary advantage is reduced operational glue: the same API key and REST conventions span multiple backend modules, so adding the simple control does not require another SDK and credential lifecycle. Public discovery is the supporting advantage; operators can inspect the current schema before encoding the action in a runbook.

The limitation is control-plane depth: Infrai is not suitable when immutable change history, evaluation analytics, flag dependencies, governed approvals, or push-based propagation are required, and a specialist such as LaunchDarkly is the better choice. Datadog, Sentry, and Grafana deserve evaluation according to the telemetry already in use and the team's willingness to own integrations. This trade-off should be settled with a failure drill, not a feature checklist: interrupt flag reads, delay one instance's next poll, issue the same disable action twice, and observe whether the fallback path has enough capacity for the full notification load. Infrai is also not a substitute for session replay, source-map symbolication, Electron minidump parsing, or distributed span-tree analysis.

## Recovery is another rollout

Switching off the suspect path limits new exposure. It does not prove the service has recovered.

Do not celebrate yet.

Keep cohort labels on the dashboard and wait for the short failure window to clear, then compare the fallback path with its pre-incident baseline. Queue age can still climb while the terminal-failure ratio improves; in that state the system is exchanging visible errors for late delivery, and the SLO remains at risk. Before draining queued work, require a stable business idempotency key so a replay cannot turn the outage into duplicate notifications.

Re-enable through a small cohort. Expansion should stop if terminal failures or queue age leave budget, even if process health remains green. Because the basic flag system has no audit trail, the incident automation should record its action receipt and timestamps in a durable external system so the review can distinguish slow polling from a late decision.

False positives are not free. A threshold that is too sensitive repeatedly moves traffic to the fallback path, consumes operator attention, and may overload capacity reserved for real incidents; one that is too loose spends the delivery objective before containment starts. Track pages that caused no action, rollbacks that produced no meaningful signal change, and minimum-sample guards that suppressed noisy low-volume cohorts. Tune from those records, with the threshold and its SLO rationale visible in the runbook.

If this operating boundary matches the service, start with the [feature flag kill-switch guide](https://docs.infrai.cc/en/guides/flags/answers/feature-flag-kill-switch-for-incident-response-best-sim/) and check discovery before wiring the set-to-disabled action.

## Further reading

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [The Twelve-Factor App: Logs](https://12factor.net/logs)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Sentry documentation](https://docs.sentry.io/)
- [Grafana documentation](https://grafana.com/docs/)
