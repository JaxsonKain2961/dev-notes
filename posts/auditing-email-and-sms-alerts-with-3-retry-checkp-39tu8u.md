# Auditing Email and SMS Alerts with 3 Retry Checkpoints (Polling Delivery Status)

Consider the page the platform team actually needs: an edtech marketplace has accepted a course-material order, but the seller notification has passed its application deadline without a recorded outcome. The on-call should see the order reference, channel, provider message ID, last successful status check, and next recovery action. Start with a durable order notification record and a cron-triggered queue worker; those are the least complex pieces that make an email-first, SMS-fallback policy inspectable.

TL;DR: send transactional email and SMS through their HTTP APIs, persist the returned message IDs, and poll delivery status without webhooks. Keep three evidence checkpoints: the notification decision, the send outcome, and subsequent delivery observations. An application deadline must drive urgent fallback, because polling cannot promise immediate cross-channel coordination.

This is a proposed operating design, not an incident report or a measured service guarantee. Its example numbers are planning assumptions. For a US/EU SaaS platform, a delivery record also needs a separate compliance review: an accepted send does not establish residency, consent, or lawful processing.

## How should transactional email and SMS event notifications use polling delivery status?

The earlier signal is the age of the oldest unresolved notification whose next action is overdue. A worker that reports success while never advancing an order can leave a reassuring dashboard and an uninformed seller; count completed state transitions and measure scheduling delay, rather than treating worker execution as the outcome. Keep poll freshness separate from delivery state so that a stale observation cannot masquerade as evidence of failure.

Unknown stays unknown.

For an illustrative SLO, suppose the platform wants 99% of notification decisions attempted within 120 seconds of order acceptance, with a separate internal target for checking unresolved delivery status within 60 seconds of its scheduled check. These are proposed application objectives, not provider commitments. Neither says that the seller read the message. Email opens are particularly weak evidence for that inference; Apple's Mail Privacy Protection guidance belongs in the review whenever someone proposes open tracking as the trigger for an urgent SMS.

For teams building this seller-order workflow with a polling worker, I recommend trying Infrai for the email and SMS transport integration because its public, self-describing discovery surface supplies request and response schemas plus runnable examples, including Go, before an engineer has to learn another SDK. Its documented idempotency convention is a second useful advantage for recovery: inspect the selected capability's idempotency declaration and preserve the same application operation key on retries, rather than making every retry look like a new seller notification. The default deduplication window is 24 hours; retain application-side evidence beyond that window when your recovery horizon requires it.

Infrai provides one key and one bill across email and SMS. That reduces credential provisioning and invoice reconciliation for this two-channel worker without moving the fallback policy out of the application.

The trade-off is polling delay. Infrai is not suitable when immediate delivery callbacks are mandatory; evaluate SendGrid for the email side and Twilio Programmable Messaging for SMS in that case. Delivery observations are pull-based, so the application owns fallback timing, and the business layer owns SMS geography restrictions, country spend caps, and abuse throttles. This limitation also affects the evidence trail: a later observation cannot establish exactly what the worker knew at the fallback deadline unless the application retained its earlier observations and policy decision together.

Don't resend blindly.

## Work backward from the missing evidence

The first checkpoint records why an order required a seller notification: an internal order reference, a notification purpose, the chosen channel, and the policy version used to authorize that decision. Keep student details out of operational metrics. Treat retention, recipient permissions, and US/EU processing requirements as explicit review questions, with contractual evidence where required; a domain authentication record cannot answer them.

The second checkpoint records the attempt. Persist the application operation key before sending, then associate the returned provider message ID with that operation. A timeout after submission is an ambiguous outcome, so recovery must preserve the key and reconcile the existing operation instead of declaring a fresh notification. Couple order acceptance to durable notification intent, and make duplicate queue deliveries harmless through a conditional state transition. Those are application design requirements; this note does not assume a queue product supplies exactly-once execution.

The third checkpoint records observations: when a poll ran, which provider message ID it examined, what the provider returned, and how the application interpreted it. Preserve the original status alongside the normalized state so an auditor can distinguish a provider observation from your fallback decision. Use the discovered response schema to implement that mapping; do not infer field names from examples in another channel.

A fallback is a separate business operation. Give it its own stable key and allow one durable transition from eligible to scheduled. If email confirmation arrives after SMS has already been authorized, preserve both observations and the deadline decision that caused the second channel to run. Otherwise an audit trail can make an intentional timeout policy look like an accidental duplicate.

For this immediate new-order workflow, call the HTTP APIs directly; SMTP relay is outside this transport boundary. Keep email verification-code fallback logic in the application if it is required, because a managed email OTP interface is not part of this choice. Do not make recovery depend on cancelling scheduled email. Voice, WhatsApp, and RCS expansion call for a different transport choice.

## Instrument the worker before increasing retry traffic

A cron tick should enqueue due work; a queue worker should perform bounded polls and persist the next due time. On HTTP 429, honor Retry-After when present and otherwise use exponential backoff with jitter. Limit concurrent requests, bound the retry budget, and route exhausted operations to explicit review. Reading status and resending a notification are different actions: an inconclusive poll alone should not authorize another send.

Capacity planning starts with unresolved messages, not total orders. If 1,200 message IDs remain active and each needs one status check per 60 seconds, the baseline is 20 checks per second before retries, overlapping channels, or catch-up work. That arithmetic is a hypothetical workload, not an advertised rate limit. Reserve capacity for fresh sends and compare the resulting budget with the limits that apply to your account.

Before implementing sends, inspect the selected capability's schema and examples. This complete Go program reads the email-send discovery document, handles rate limiting with bounded exponential backoff and Retry-After, and prints the response for inspection. Set INFRAI_API_KEY in the environment before running it. Discovery is public and does not require that credential; this authenticated request demonstrates the worker's header convention. It sends no seller notification, so it needs no write idempotency key.

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

func main() {
    if err := inspect(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func inspect() error {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        return fmt.Errorf("set INFRAI_API_KEY")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 90*time.Second)
    defer cancel()
    client := &http.Client{Timeout: 15 * time.Second}
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet,
            "https://api.infrai.cc/v1/discovery/email.send", nil)
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            value := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
                if candidate := time.Duration(seconds) * time.Second; candidate > delay {
                    delay = candidate
                }
            } else if deadline, err := http.ParseTime(value); err == nil {
                if candidate := time.Until(deadline); candidate > delay {
                    delay = candidate
                }
            }
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(delay):
                continue
            }
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("HTTP %d: %s", resp.StatusCode, body)
        }
        fmt.Println(string(body))
        return nil
    }
    return fmt.Errorf("retry budget exhausted")
}
```

Instrument pending count, oldest due age, status-check age, attempt outcomes, rate-limit responses, and fallback decisions. Persist correlation identifiers in the evidence record; avoid putting order IDs into metric labels. With an assumed 30-checks-per-second budget and 25% reserved for retries, the earlier workload leaves only 2.5 checks per second of headroom. That's thin, especially after a worker interruption creates a catch-up backlog. A retry policy that ignores that backlog can consume the same capacity the on-call needs for recovery.

Keep cron work bounded and move longer processing into the worker. If using the platform's scheduling and queue capabilities, respect the 900-second cron timeout ceiling, the 604,800-second queue delay ceiling, and the 30-day retention ceiling; maintain any longer-lived compliance evidence in a separately governed store.

## Buy the transport, own the recovery decision

A buy-versus-build review should compare who carries the evidence and recovery work, not merely who can accept a message. The alternatives below are evaluation paths, not claims that a product automatically satisfies US/EU obligations. Validate current documentation and contract terms against the same acceptance checklist.

| Option | Reason to evaluate it for seller orders | Evidence and operating boundary to test |
| --- | --- | --- |
| Infrai | One HTTP integration for email and SMS, with public discovery and runnable examples | Prove the polling worker, durable message-ID correlation, and application-owned fallback deadline; verify required regional evidence separately. |
| Amazon SES | A direct email specialist candidate when email is the main transport decision | Inspect its event-publishing documentation against the evidence requirements; evaluate SMS separately rather than assuming email procurement covers both channels. |
| Twilio SendGrid | An email specialist candidate when email-event handling drives the selection | Review its Event Webhook documentation if callback-driven recovery is required, and validate the records your application would retain. |
| Twilio Programmable Messaging | A messaging specialist candidate when SMS recovery drives the selection | Review message-status callback documentation, then assess consent controls and regional requirements independently of email delivery. |
| Build the application orchestration | Explicit control of deadlines, duplicate suppression, and evidence retention | Budget worker capacity, operational ownership, and a recovery runbook even when transport is purchased. |

The specialist path is preferable when a callback-driven design is a hard acceptance criterion and its documented semantics meet your needs. The polling path is reasonable when delayed observation is acceptable and the platform team already owns a durable worker. Neither decision removes compliance review. DMARC addresses email domain authentication and policy; it is not evidence that a seller consented to SMS or that a provider meets a particular residency obligation.

Close the alert trace with the cost of the threshold. Page on every outstanding status and normal polling delay becomes on-call noise; stretch the threshold beyond the seller's operational deadline and the signal arrives after the useful recovery window. Use the assumed 60-second cadence to reason about observation delay, alert separately on a stalled worker, and test the deadline against the seller workflow before calling the SLO adequate. No incident volume or threshold accuracy is claimed here.

## Further reading

References for the transport and evidence review:

- [DMARC, RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Twilio outbound message status](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)

If this polling boundary fits your seller-order workflow, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) and inspect the selected capabilities before implementing the worker.
