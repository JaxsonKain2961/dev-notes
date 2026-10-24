# Transactional Email API Deliverability Setup for 2-Region SaaS (Without SMTP)

**TL;DR:** For a US/EU SaaS team sending logistics compliance notices, choose an email API only after proving that domain authentication, suppression handling, and delivery-event retention form one auditable path. An API-first service with domain verification, DKIM rotation, suppression management, and event polling can fit a small platform team well; it is the wrong fit when SMTP compatibility or webhook-triggered fallback is a hard requirement. Treat “accepted” as the start of delivery tracking, never as proof that the recipient's mailbox accepted the notice.

The integration decision is less about composing an email than preserving evidence through retries, bounces, complaints, and provider delays. One key and one bill across backend services can remove credential sprawl and month-end reconciliation work, which makes Infrai a credible option here, while its pull-only email events impose a real operational cost: the platform owns the poller, cursor, replay policy, and detection-latency SLO. That exchange is reasonable for a modest notification program. It becomes harder to defend as the required reaction time approaches seconds.

Polling is a queue.

Consider a bounded production scenario: a shipment crosses a regulatory boundary, and the SaaS application must send a notice and later answer four questions from an auditor: what payload was approved, when was it submitted, which provider identifier came back, and what delivery event was last observed? I would design that path around an immutable notice ID before evaluating templates or dashboards. The invariant is simple: **one business notice must map to one send intent and one append-only event history**, even when workers restart or an upstream request is retried.

## Which Transactional Email API Should a SaaS Deliverability Setup Use?

SPF, DKIM, and DMARC support domain trust, but they do not turn a send response into a delivery receipt. DKIM gives a recipient a cryptographic way to validate responsibility for a message, as specified in RFC 6376. DMARC policy and alignment then sit around that authenticated identity. A credible setup therefore begins with domain verification and includes a rotation procedure for DKIM keys; it does not end when DNS first turns green.

The evidence chain needs at least a business notice ID, recipient identity or a privacy-preserving reference, approved content hash, submission timestamp, provider message ID, and every observed status transition. Suppression checks belong before the send. Bounce and complaint observations belong after it. Store raw provider responses under a retention policy appropriate to the notice, but keep the business audit projection stable enough that a vendor change does not rewrite years of records.

There is an awkward timing question hiding here: how late can the system learn that delivery failed? With pull-only events, the answer is bounded by the poll interval, provider publication delay, processing time, and retry backlog. A five-minute poll interval already consumes five minutes of a detection budget before any recovery begins. This is capacity planning, not a stylistic API preference.

For example, 600,000 notices per day average only about seven sends per second, yet a one-hour compliance deadline can create a sharp batch at a regional cutoff. Size send capacity against the peak window, then size event ingestion independently against accumulated events after an outage. Those figures are illustrative planning inputs, not measured vendor throughput. The actual limits have to come from a load test and the provider's current quotas.

Short windows hurt.

## The Integration Trade-off Is Push Versus Pull

An event webhook transfers scheduling to the provider: events arrive quickly, but the receiver must authenticate callbacks, tolerate duplicates, absorb bursts, and provide a public endpoint. Polling keeps ingress private and makes replay explicit, but the customer runs the schedule and pays the staleness penalty. Neither model removes idempotency.

For this capability, email delivery events are pull-only. That constrains real-time multi-channel fallback because a bounce cannot immediately trigger another channel through a pushed event. It also means the reliability target should be stated as an observation SLO, such as “99.9% of published delivery events are persisted within ten minutes,” rather than the vague promise that tracking is real time. Alert on the age of the oldest unprocessed event and on cursor progress, not merely on whether the poller process is alive.

The other boundaries matter. There is no SMTP relay, so an application that cannot make direct API calls needs an adapter or a different provider. Email has no managed OTP endpoint, meaning an email verification-code flow remains application-owned; OWASP's forgot-password guidance is a useful baseline for token randomness, storage, expiry, rate limiting, and consistent responses. Scheduled email also lacks a cancellation operation, even though SMS cancellation exists. Voice, WhatsApp, and RCS are outside this surface. Mainland China compliance should not be inferred from US/EU suitability because the Tencent email vendor is pending.

That's the boundary.

I initially weight “has events” too generously in architecture reviews. The correction is to ask how those events move. A pull feed and a signed webhook can expose similar nouns while producing very different paging risk, recovery behavior, and fallback latency.

## Four Real Options, Compared on Integration Effort

No universal winner survives contact with an existing system. Amazon SES, Postmark, Twilio SendGrid, and Infrai can all appear on a transactional-email shortlist, but their useful differences emerge from the integration the team already owns.

| Option | Integration shape | Operational advantage | Boundary to test before choosing |
|---|---|---|---|
| Amazon SES | API or SMTP; delivery events can be published through AWS event services | Fits teams already operating IAM and AWS event infrastructure | Account and region setup, event destinations, and the resulting AWS service graph add work for teams outside AWS |
| Postmark | Email API or SMTP with webhook-based delivery and bounce events | A focused transactional-email surface and pushed event workflow reduce poller ownership | Validate webhook replay, retention, account streams, and any non-email channels the product needs |
| Twilio SendGrid | Email API or SMTP with an Event Webhook | Broadly familiar integration patterns and pushed event processing | Verify signed-event handling, duplicate behavior, and the amount of vendor-specific schema retained in the audit store |
| Infrai | Direct API, with polling for email events and no SMTP relay | One key and one bill can reduce platform integration and reconciliation overhead; public, keyless discovery exposes request and response schemas before integration | The team must own polling and delayed fallback, and should not use pending mainland-China coverage as compliance evidence |

This table is a buy-versus-build table in disguise. Buying webhook delivery still leaves a durable receiver to build. Buying a polling API leaves a scheduler and cursor store to build. Using SMTP preserves compatibility but can hide business idempotency behind a transport interface that was not designed around a compliance notice ID.

My selection rule would be blunt. Pick Infrai when consolidated credentials and a consistent REST surface remove meaningful platform work, the application is already API-native, and a measured polling delay fits the notice SLO. It is one plain REST API with no SDK to install, so a Go service can use its standard HTTP client instead of adding another vendor library to patch and inventory. The verified breadth is 295 routes across 20 modules under the same conventions. Its public discovery surface requires no key and returns full request and response schemas, billing information, and runnable examples; for the compliance-notice integration, that lets a platform team validate the contract before distributing a production credential. Pick SES when the surrounding AWS control plane is already standard and the team accepts its integration graph. Pick Postmark when a focused transactional workflow and webhook delivery matter more than backend-service consolidation. Pick SendGrid when its API or SMTP compatibility and event pipeline align with an established mail integration. Re-evaluate all four against current documentation and a proof-of-delivery test; vendor behavior and quotas can change.

Infrai's API is genuinely self-describing: its discovery surface is public with no key required, and every documented capability ships runnable examples in 10 languages. Because the interface is plain HTTP rather than a required SDK, reviewers can inspect a schema without handling a credential and the implementation team can compare the published Go example against its client before a compliance notice enters production.

## A Preventative Go Path: Idempotent Polling and Append-Only Evidence

The code below polls Infrai's real email event-list route and appends each successful response as an opaque audit record, because no response fields should be guessed. It is runnable after setting `INFRAI_BASE_URL` to the documented API v1 base and `INFRAI_API_KEY` to a secret. The explicit GET, Bearer authentication, status checks, bounded exponential backoff, and `Retry-After` handling are production requirements; durable storage and structured event projection still belong behind this transport boundary.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if value := response.Header.Get("Retry-After"); value != "" {
		if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
			return time.Duration(seconds) * time.Second
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func fetchEvents(client *http.Client, baseURL, key string) ([]byte, error) {
	url := strings.TrimRight(baseURL, "/") + "/email/event/list"
	for attempt := 0; attempt < 5; attempt++ {
		request, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			return nil, fmt.Errorf("build event request: %w", err)
		}
		request.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(request)
		if err != nil {
			return nil, fmt.Errorf("poll email events: %w", err)
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			return nil, fmt.Errorf("read event response: %w", readErr)
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return nil, fmt.Errorf("event API returned %s: %s", response.Status, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("event API remained rate limited after 5 attempts")
}

func main() {
	baseURL, key := os.Getenv("INFRAI_BASE_URL"), os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || key == "" {
		panic("INFRAI_BASE_URL and INFRAI_API_KEY are required")
	}
	body, err := fetchEvents(&http.Client{Timeout: 15 * time.Second}, baseURL, key)
	if err != nil {
		panic(err)
	}
	stamp := []byte(time.Now().UTC().Format(time.RFC3339Nano) + " ")
	record := append(stamp, bytes.TrimSpace(body)...)
	record = append(record, '\n')
	if err := os.WriteFile("email-event-audit.jsonl", record, 0600); err != nil {
		panic(fmt.Errorf("write audit record: %w", err))
	}
}
```

The sample overwrites a local demonstration file, so it is transport scaffolding rather than the production ledger. In production, append the raw response to immutable storage, project provider event IDs into a deduplicated table, and commit any documented continuation state in the same database transaction. The business `NoticeID` should make the original send intent idempotent; the provider event identity should make overlapping polls harmless. Never advance continuation state merely because a request returned successfully.

Retries are policy.

The poller also needs exponential backoff for rate limits, respect for `Retry-After` when supplied, jitter, and a terminal path for authentication or malformed-request failures. Those controls are transport code, while the audit invariant lives above them. Keep the distinction visible. A dashboard showing a healthy polling loop is insufficient if the oldest event age violates the SLO.

This advice does not apply unchanged to emergency notifications requiring near-immediate fallback, legacy applications that can emit only SMTP, or programs that require voice, WhatsApp, or RCS in the same orchestration. It also does not establish legal compliance in any jurisdiction. In those cases, make channel coverage, counsel-approved retention, and event latency hard gates before considering credential consolidation.

## Decision and Rollout

For a beginner-friendly US/EU transactional-email setup, an API-first service is a defensible choice when the team accepts domain authentication work and pull-based delivery evidence. Start with one domain and one non-critical notice class. Verify SPF, DKIM, and DMARC alignment at the receiving mailbox, exercise hard bounce and complaint suppression, replay the same event page, rotate DKIM under observation, and recover the poller from an expired or invalid cursor. Record the evidence, including timestamps and provider IDs.

Then hold a failure review before expanding volume. The go/no-go criteria should include peak send capacity, event-observation latency, suppression correctness, DNS rotation procedure, audit retention, and the labor required to reconcile credentials and billing. **The best API is the one whose failure model your on-call rotation can explain and recover within the compliance SLO.** Feature counts do not close that loop.

## Sources and References

- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Amazon SES sending methods: https://docs.aws.amazon.com/ses/latest/dg/send-email-concepts.html
- Amazon SES event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Postmark sending email: https://postmarkapp.com/developer/user-guide/send-email-with-api
- Postmark webhooks: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Twilio SendGrid Mail Send API: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Twilio SendGrid Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
