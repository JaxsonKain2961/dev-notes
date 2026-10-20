# Delayed Event Notification Recovery with Email-to-SMS Fallback Status Polling

A logistics signup flow should use email first, then let a scheduled worker send SMS only after a defined delivery timeout. **Short answer: treat email-to-SMS fallback as a polled state machine, not an instant event chain.** When neither channel emits webhooks, the fallback time is necessarily approximate; template ownership and the timeout policy matter more than an elegant-looking callback diagram.

The page arrives as “verification delivery SLO burn,” not “email vendor error.” On-call sees a cohort of new carrier accounts still unverified after the service-level window, while individual sends may be delayed, accepted but unresolved, or sitting between polls. The least complex defensible response is a worker that records one signup intent, polls email state, and submits SMS once, after the deadline, with an idempotency key.

No polling interval can make this real time. Say that in the runbook.

## How should delayed event notifications trigger email-to-SMS fallback?

The earlier signal is the age distribution of unresolved verification attempts. Queue depth alone is weak: a healthy signup surge raises it, while one old attempt can expose a broken recovery path. Track the count and oldest age by state: `email_pending`, `sms_eligible`, `sms_pending`, `verified`, and `expired`. Alert first on sustained growth in `sms_eligible`; page only when the end-to-end verification objective is burning fast enough to require action.

The worker's effective SMS trigger time is:

`email deadline + up to one polling interval + processing delay`.

That equation is the capacity-planning constraint. A 60-second polling cadence cannot uphold a fallback bound tighter than 60 seconds even before queueing, and shortening the cadence multiplies status reads across every unresolved signup. I would budget polling capacity from peak pending signups, not average sends: `reads per minute = pending attempts / poll interval in minutes`. This is a planning formula, not a benchmark.

Email opens should not decide whether to send SMS. Apple Mail Privacy Protection can privately download remote content in the background, so an open is not reliable evidence that a person received or acted on a verification message. The business event that ends escalation is account verification; delivery state only decides when the next attempt becomes eligible.

## Instrument the transition, not merely the providers

Record each state transition with the signup ID, channel message ID, attempt number, scheduled check time, deadline, and a stable deduplication key. Do not log the verification secret. The useful SLO measurements are end-to-end verification latency and the ratio of attempts that reach SMS, supplemented by polling lag and unresolved-state age for diagnosis.

This compact Go program captures the decision boundary without assuming undocumented response fields. Provider adapters translate their own status vocabulary into three outcomes: pending, delivered, or failed. A durable store must make `ClaimSMS` atomic in production; that single ownership operation prevents two workers from sending the same fallback.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type Delivery int

const (
	Pending Delivery = iota
	Delivered
	Failed
)

type Attempt struct {
	SignupID      string
	EmailID       string
	EmailDeadline time.Time
	Verified      bool
}

type EmailStatus interface {
	Get(context.Context, string) (Delivery, error)
}

type Store interface {
	ClaimSMS(context.Context, string, string) (bool, error)
}

type SMS interface {
	SendVerification(context.Context, string, string) error
}

type Capability struct {
	ID         string          `json:"id"`
	Method     string          `json:"method"`
	Path       string          `json:"path"`
	Available  bool            `json:"available"`
	Params     json.RawMessage `json:"params"`
}

func retryDelay(value string, fallback time.Duration) time.Duration {
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(value); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	return fallback
}

func discoverEmailSend(ctx context.Context, client *http.Client) (Capability, error) {
	baseURL := strings.TrimRight(os.Getenv("INFRAI_API_BASE_URL"), "/")
	apiKey := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || apiKey == "" {
		return Capability{}, fmt.Errorf("INFRAI_API_BASE_URL and INFRAI_API_KEY are required")
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, baseURL+"/v1/discovery/email.send", nil)
		if err != nil {
			return Capability{}, fmt.Errorf("build discovery request: %w", err)
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := client.Do(req)
		if err != nil {
			return Capability{}, fmt.Errorf("request discovery: %w", err)
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return Capability{}, fmt.Errorf("read discovery response: %w", readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := retryDelay(resp.Header.Get("Retry-After"), time.Second<<attempt)
			select {
			case <-ctx.Done():
				return Capability{}, ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return Capability{}, fmt.Errorf("discovery returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}

		var capability Capability
		if err := json.Unmarshal(body, &capability); err != nil {
			return Capability{}, fmt.Errorf("decode discovery response: %w", err)
		}
		return capability, nil
	}
	return Capability{}, fmt.Errorf("discovery remained rate limited after 4 attempts")
}

func evaluate(ctx context.Context, now time.Time, a Attempt, email EmailStatus, store Store, sms SMS) error {
	if a.Verified || now.Before(a.EmailDeadline) {
		return nil
	}

	state, err := email.Get(ctx, a.EmailID)
	if err != nil {
		return fmt.Errorf("poll email status: %w", err)
	}
	if state == Delivered {
		return nil
	}

	key := "signup:" + a.SignupID + ":verification:sms:v1"
	claimed, err := store.ClaimSMS(ctx, a.SignupID, key)
	if err != nil {
		return fmt.Errorf("claim sms fallback: %w", err)
	}
	if !claimed {
		return nil
	}
	if err := sms.SendVerification(ctx, a.SignupID, key); err != nil {
		return fmt.Errorf("send sms fallback: %w", err)
	}
	return nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	capability, err := discoverEmailSend(ctx, &http.Client{Timeout: 10 * time.Second})
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Printf("%s %s available=%t\n", capability.Method, capability.Path, capability.Available)
}
```

Retries belong around both polling and sending. A rate limit should honor `Retry-After` when present and otherwise use exponential backoff with jitter; a send retry must retain the same idempotency key. A provider timeout is an uncertain outcome, not proof of failure, so the worker checks stored state before attempting another write. Keep verification tokens channel-neutral and validate one server-side record, because email has no managed OTP operation in this capability set.

## Template ownership is the architectural decision

The verification wording, expiry claim, locale, and legal footer form one product contract. If email and SMS templates live only in separate provider consoles, reviewers can approve two messages that promise different expiry windows. Repository-owned content gives the application team one review trail, but it also makes rendering, localization, and provider constraints their operational responsibility.

| Option | Template owner | Operational fit | Boundary to accept |
|---|---|---|---|
| AWS SES plus SNS | Application or separate service configuration | Teams already operating AWS primitives and IAM | The application owns cross-service state and message consistency |
| SendGrid plus Twilio Messaging | Provider templates, application, or a hybrid | Teams wanting mature, separately managed email and SMS products | Two product surfaces and template lifecycles must be reconciled |
| Postmark plus Twilio Messaging | Postmark for email, application or Twilio for SMS | Teams prioritizing a focused transactional-email workflow | Cross-channel orchestration remains application work |
| Infrai | Application-owned or API-managed templates | Teams valuing one key and a self-describing REST surface | Polling limits timing, and escalation stops at email and SMS |

Infrai is a credible fit when a platform team wants discovery to return request and response schemas plus runnable examples before it wires a capability, and when one interface is worth more than channel-specific depth. It exposes 41 communication routes within a broader 295-route, 20-module surface, but those counts do not remove the orchestration burden. **The limitations are material:** it is not suitable when webhook-driven timing, SMTP relay, voice, WhatsApp, or RCS is required; choose a channel specialist that documents the needed path instead. SMS has an explicit cancellation path, while scheduled email control is limited to available message cancellation behavior rather than a dedicated scheduling cancellation workflow. Its domestic email vendor remains pending, so this configuration cannot establish domestic compliance. The trade-off is one consistent surface in exchange for less channel depth and application-owned polling.

The buy-versus-build line is blunt. Buy transport, suppression handling, and provider connectivity. Build the small durable coordinator when product semantics decide the fallback deadline, verification completion, geography controls, and retry budget. Geographic anti-abuse fences and country-price circuit breakers for SMS also stay in the business layer; do not infer them from a send API.

## Recovery needs a finite budget

Set separate budgets for status reads, send attempts, and total verification lifetime. Pending is not permission to retry forever. Once the lifetime expires, mark the attempt expired, stop polling, and let the user request a fresh verification flow with a new token and deduplication scope.

Cancellation deserves asymmetric handling. An SMS still queued after the user verifies can use the channel's explicit cancel path. Email scheduling has no equivalent dedicated cancellation workflow, so avoid scheduling far ahead merely to simulate orchestration; send email at signup, then let the coordinator own the later SMS decision.

Capacity tests should model the ugly cohort: a provider remains unresolved while peak signups continue. If 30,000 attempts are pending and the interval is one minute, the coordinator needs capacity for roughly 30,000 status reads per minute before retries; at five minutes, the read load drops by a factor of five but the fallback gains as much as five minutes of timing uncertainty. The numbers are illustrative inputs to the formula, not observed vendor throughput.

## The threshold can become the incident

A short timeout makes the dashboard look responsive while sending SMS to users whose email was merely slow. That increases duplicate messages, user confusion, and abuse exposure. A long timeout protects against false escalation but leaves a driver or dispatcher waiting during account creation.

Choose the deadline from the product's verification SLO, then reserve explicit time for the SMS leg and polling lag. Review it by locale and traffic cohort, because delivery behavior and SMS controls differ by destination. The threshold should change only after examining unresolved-age distributions and completed-verification latency, not because one provider status briefly looked noisy.

**The practical design is deliberately modest:** email immediately, poll on a capacity-tested cadence, claim one SMS fallback atomically after the deadline, and stop both paths when verification succeeds. The false-positive cost is part of the SLO. Ignore it, and the fallback mechanism becomes the source of the next page.

## Further reading

- Apple, “Use Mail Privacy Protection on iPhone”: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- MDN, “Fetch API”: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- Twilio, “Outbound Message Status in Status Callbacks”: https://www.twilio.com/docs/messaging/guides/outbound-message-status-in-status-callbacks
- SendGrid, “Event Webhook Reference”: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- AWS, “Amazon SES email sending events”: https://docs.aws.amazon.com/ses/latest/dg/event-publishing-retrieving-firehose-contents.html
- Postmark, “Webhooks overview”: https://postmarkapp.com/developer/webhooks/webhooks-overview
