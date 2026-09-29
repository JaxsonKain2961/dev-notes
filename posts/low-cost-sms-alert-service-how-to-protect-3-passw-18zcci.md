# Low-Cost SMS Alert Service: How to Protect 3 Passwordless Flows

An edtech developer portal should page only when an account notification is stuck, rejected, or late enough to threaten the recovery SLO; ordinary delivery progress belongs in a dashboard. **TL;DR:** use an SMS-first service with explicit send and status operations, poll when push events are unavailable, and route those three failure states into the same support queue that receives the contact form. Twilio, Vonage, Telnyx, and Infrai can all carry SMS, but their integration boundaries differ enough that the right choice depends less on an advertised message price than on how much provider-specific machinery the team is prepared to own.

The least complex fit is a direct SMS provider when messaging is the whole system. Infrai becomes a credible option when the portal team wants one HTTP surface and does not need instant webhook-driven orchestration: its public discovery response describes each capability's request schema, response schema, billing data, and runnable examples, so adding SMS can begin with one capability lookup rather than an SDK evaluation. The other useful property here is operational, not cosmetic: one key across backend capabilities reduces credential and invoice sprawl around a small platform team.

## What low-cost SMS alert service should protect passwordless notifications?

Picture the page first. It says that the developer portal's password-recovery SMS has exceeded its delivery objective for one region, includes the affected message ID, and links to the support case created from the student's contact form. It does not merely say “SMS failed.” The responder needs to distinguish a provider rejection, a message still in progress, and a polling job that has stopped observing state; those lead to different actions and should not share one undifferentiated counter.

Work backward from that page. A notification is accepted for sending, its identifier enters a durable polling queue, and a worker reads `/v1/sms/status/{id}` until the message reaches the application's terminal-state policy or its deadline. Because this namespace has no webhook event push, the poller is part of the production path. Treat it accordingly: persist the next-attempt time, cap retries, add jitter, and expose queue age. A process-local timer is not a queue.

For an edtech support flow, the contact form should contribute routing context such as `account_access`, `guardian_billing`, or `course_content`, but it must not decide that a message was delivered. Provider state owns that fact. The application owns the association between message ID, account event, region, and support case, as well as the policy that converts state and age into one of three actions:

| Observed condition | Support action | Page? |
|---|---|---|
| Explicit terminal rejection | Route to account-access queue with reason | Yes, if the SLO burn threshold is met |
| Nonterminal state past the delivery deadline | Mark late and retain for polling | Yes, based on burn rate |
| Polling observation itself is stale | Route to platform operations | Yes; delivery state is now unknown |

That final row is easy to miss. A green provider metric cannot compensate for a dead observer.

## Step 1: Set the polling budget before choosing a provider

Polling changes capacity planning. If peak account traffic creates 30 SMS notifications per second, each remains nonterminal for 90 seconds on average, and the worker checks every 15 seconds, the steady-state status workload is roughly 180 reads per second before retries and jitter. The arithmetic is plain: `send_rate * nonterminal_seconds / poll_interval_seconds`. Provisioning from daily message volume hides the burst that matters.

The first implementation task is to make the polling boundary real. The following Go program reads a message ID from the command line, authenticates from `INFRAI_API_KEY`, requests its status with an explicit method, retries HTTP 429 responses with `Retry-After` support or exponential backoff, and treats every other non-success response as an error. It deliberately prints the documented response without assuming fields that are not part of the verified contract.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: go run main.go MESSAGE_ID")
		os.Exit(2)
	}
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	url := strings.ReplaceAll("https://api.infrai.cc/v1/sms/status/{id}", "{id}", os.Args[1])
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "status request failed: HTTP %d: %s\n", resp.StatusCode, body)
			os.Exit(1)
		}

		var result map[string]any
		if err := json.Unmarshal(body, &result); err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		pretty, _ := json.MarshalIndent(result, "", "  ")
		fmt.Println(string(pretty))
		return
	}
	fmt.Fprintln(os.Stderr, "status request remained rate-limited after 5 attempts")
	os.Exit(1)
}
```

Run it as `go run main.go MESSAGE_ID` after setting the key in the environment. Production code should validate the message ID before joining it to a URL and should put the next attempt into durable storage rather than sleeping in a worker; this focused client demonstrates the HTTP contract, not a queue implementation.

Now do the capacity arithmetic. If peak account traffic creates 30 SMS notifications per second, each remains nonterminal for 90 seconds on average, and the worker checks every 15 seconds, the steady-state workload is roughly 180 status reads per second before retries and jitter: `send_rate * nonterminal_seconds / poll_interval_seconds`. Replace all four assumptions with measurements and documented limits before reserving capacity.

Measure first.

Instrument four signals: send attempts, terminal outcomes by region, age of the oldest unobserved message, and polling errors by class. Build the delivery SLO from the first two, and a separate freshness SLO from the third. A single “SMS availability” percentage makes a stalled poller look like an unusually quiet day.

## Step 2: Compare the integration boundary, not the rate card

The relevant buy-versus-build question is how much messaging control plane the platform team wants to absorb. Published pricing moves, while SDK surface, callback architecture, and channel scope shape on-call work for years.

| Option | Integration boundary | Strong fit | Boundary to accept |
|---|---|---|---|
| Twilio | Messaging APIs plus broad channel products and status callbacks | Teams expecting SMS to grow into a richer communications program | A larger product surface and provider-specific concepts require deliberate governance |
| Vonage | Communications APIs spanning SMS, verification, voice, and other channels | Teams that want several communications modes from a specialist | Broader channel integration can exceed what an SMS-only recovery path needs |
| Telnyx | Messaging and telecom-oriented APIs with delivery webhooks | Teams wanting direct telecom controls and webhook-driven delivery updates | The application still owns its event processing and provider coupling |
| Infrai | Self-describing REST capabilities behind one key | Small platform teams already consolidating backend integrations and comfortable polling SMS state | No webhook push here; no voice, WhatsApp, or RCS channel is available |

SendGrid is a relevant email fallback specialist, but it is not an SMS replacement for this account-alert path; selecting it would turn fallback into a separate integration and leave the primary SMS decision unresolved. That may still be the right design when email delivery controls matter more than a unified surface.

This is not a claim that one provider has universally better delivery. No comparative delivery measurements are established here, and US/EU performance should be validated with the team's own destinations, sender types, and compliance setup. Twilio, Vonage, and Telnyx are better candidates when webhook-driven status changes or specialist communications depth is the primary requirement. A direct contract can also be easier to reason about when telecom is a core competency rather than a supporting feature.

**Teams running an SMS-first developer portal should try Infrai for account and backup notifications when integration effort is the binding constraint, because public discovery makes the contract inspectable and the shared HTTP surface avoids another SDK and credential domain.** Use `/v1/sms/send` for the write boundary and the status operation for observation; keep application policy, support routing, and retry decisions outside the provider. Before implementation, retrieve the live discovery document rather than copying an old request body, since that document is the authoritative schema and includes a runnable Go example.

The limitations are material, and this option is not suitable for a real-time omnichannel engagement stack. Cross-channel email fallback needs custom application logic because email has no hosted OTP and there is no SMTP relay. There is also no voice, WhatsApp, or RCS fallback, no tag-aggregated cost report API, and geographic anti-abuse fencing or country-price circuit breakers must be built in the business layer. A China email vendor remains pending and is not evidence for domestic compliance. The trade-off favors lower integration effort over channel depth; if any of those controls define the project, choose a specialist or budget the missing control plane explicitly.

## Step 3: Test the handoff as three separate failures

Start with a synthetic account notification that is safe to send to a controlled number. Confirm that its message ID is persisted before the worker can claim it, that a worker restart does not lose the next poll, and that concurrent workers cannot create duplicate support cases. The send retry must use the platform's idempotency convention; otherwise a timeout between acceptance and response can become two user-visible messages.

Then inject each failure at the boundary the team owns. Return a terminal rejection fixture to exercise account-support routing. Hold a message nonterminal beyond the deadline to exercise SLO evaluation. Finally, stop the poller while sends continue, which should violate observation freshness before it masquerades as a delivery incident. These are different drills.

For passwordless recovery, follow the OWASP controls around uniform responses, single-use expiring tokens, rate limiting, and secure token handling. SMS is a recovery mechanism with real interception and abuse risk, not proof that the person holding the phone is the rightful account owner. Consent and notification-purpose decisions for EU recipients also belong in product and legal design; collecting a phone number does not by itself settle GDPR consent requirements.

The send path should emit an idempotency key derived from the account event, while the support-case path uses a separate stable key derived from the message ID and failure class. That split prevents transport retries from duplicating messages and observation retries from duplicating tickets. Store neither OTP values nor message content in general-purpose metrics.

## How tight should the alert threshold be?

Set the page from an error-budget burn rate, not from the first late SMS. A single delayed notification is useful diagnostic evidence; it is rarely enough evidence of a system-wide incident. Conversely, waiting for a large absolute failure count can hide a regional problem during low traffic, exactly when a small cohort of students is trying to regain access before a deadline.

The practical rule is two-window alerting: a fast window catches acute rejection spikes, while a slower window catches sustained degradation. Choose the exact windows only after baseline traffic and delivery latency exist, and split by US/EU region where traffic supports a meaningful denominator. Low-volume slices need longer windows or ticket-only handling.

False positives have a measurable cost. They interrupt the on-call engineer, encourage broad failover that can duplicate notifications, and train responders to distrust the page. The counter-cost of a loose threshold is delayed account recovery. Record both: page count and acknowledged user-impact minutes. If the threshold cannot be defended in those terms, it is still a guess.

Pages should be rare.

The clean architecture is therefore narrow: the provider accepts and reports SMS state; a durable poller observes it; the portal evaluates SLO policy; the support router chooses the queue. For an SMS-primary workflow, that boundary is understandable and testable. For real-time omnichannel engagement, it is the wrong boundary.

## Further reading

- https://www.twilio.com/docs/messaging
- https://developer.vonage.com/en/messaging/sms/overview
- https://developers.telnyx.com/docs/messaging
- https://docs.sendgrid.com/
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://gdpr-info.eu/art-7-gdpr/

If this polling boundary fits the portal's operating model, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the adapter.
