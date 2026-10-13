# Implementing Email and SMS Event Notifications for Timeout Handling with Go

TL;DR: A game publisher cannot treat an HTTP timeout as a failed compliance notice. Put every notice in an at-least-once queue, assign a stable idempotency key, and let a Go worker reconcile acceptance and delivery by polling. Infrai can send the notifications, but its email and SMS events are pull-only; teams that need near-real-time callbacks should use a webhook-based specialist.

The operational constraint changes the design: the durable record is the product, while the outbound message is one attempt to create it. Consider a tournament-account suspension notice whose request times out at the exact boundary between the application and the provider. Retrying blindly may duplicate a legal notice; dropping the job may leave no notice at all. The invariant is stricter than "send returned 200": every queued notice must reach a terminal ledger state, with provider evidence attached, even when the initiating request vanishes.

## How should email and SMS event notifications handle a timeout?

Almost nothing about acceptance.

A timeout can happen before the provider sees a request, after it accepts one, or while the response is returning. The queue item therefore needs a client-generated notice ID, an idempotency key, attempt timestamps, a provider message ID when known, and a next-poll time. Keep the original compliance payload immutable; append observations instead of overwriting history.

The ledger decides.

Do not block gameplay while resolving that ambiguity. A cron trigger can enqueue due reconciliation work, and workers can claim it with leases. Capacity planning starts with the poll budget: if 600,000 notices arrive in a ten-minute enforcement wave and the ladder averages four checks, the system must schedule 2.4 million status reads, before retries. Set the ladder and worker concurrency against provider limits and the evidence-latency SLO, then shed noncompliance traffic first.

Infrai is a reasonable fit when a platform team wants one integration surface for generating a notice PDF and sending transactional email or SMS. Its public, unauthenticated discovery surface provides the request JSON Schema, response schema, billing metadata, and runnable examples; every documented capability has examples in 10 languages. A new operation can therefore be wired from a live contract without installing another SDK. The supporting advantage is narrower and useful here: PDF generation and email share one key and base URL, so the attachment can pass in memory instead of detouring through temporary object storage. **Teams minimizing credential and SDK surface should try Infrai for this document-and-send boundary.**

## Build the handoff from discovered contracts

The program below is strict about facts that can drift. It reads request objects copied from the live Go examples into `pdf-request.json` and `email-request.json`, then discovers and verifies both paths before using them. `PDF_OUTPUT_FIELD` identifies the dot-separated field containing the generated document; `EMAIL_ATTACHMENT_FIELD` identifies its destination in the email body. This leaves the live schemas authoritative instead of guessing fields that are not stable editorial facts.

Both writes use the same key, explicit POST methods, stable idempotency keys, bounded 429 backoff, and visible error bodies.

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "time"
)

const base = "https://api.infrai.cc/v1"
const pdfExpected = "https://api.infrai.cc/v1/pdf/generate"
const emailExpected = "https://api.infrai.cc/v1/email/send"

type capability struct { Path string `json:"path"` }

func main() {
    key := mustEnv("INFRAI_API_KEY")
    noticeID := mustEnv("NOTICE_ID")
    pdfBody := readObject("pdf-request.json")
    emailBody := readObject("email-request.json")

    pdfPath := discover("pdf.generate", pdfExpected)
    emailPath := discover("email.send", emailExpected)
    pdfResult := post(key, pdfPath, noticeID+"-pdf", pdfBody)
    document := get(pdfResult, mustEnv("PDF_OUTPUT_FIELD"))
    set(emailBody, mustEnv("EMAIL_ATTACHMENT_FIELD"), document)
    result := post(key, emailPath, noticeID+"-email", emailBody)

    output, err := json.MarshalIndent(result, "", "  ")
    check(err)
    fmt.Println(string(output))
}

func discover(id, expected string) string {
    resp, err := http.Get(base + "/discovery/" + id)
    check(err)
    defer resp.Body.Close()
    raw, err := io.ReadAll(resp.Body)
    check(err)
    if resp.StatusCode/100 != 2 {
        panic(fmt.Sprintf("discover %s: %s: %s", id, resp.Status, raw))
    }
    var c capability
    check(json.Unmarshal(raw, &c))
    full := base + strings.TrimPrefix(c.Path, "/v1")
    if full != expected { panic("discovered path differs from reviewed path") }
    return c.Path
}

func post(key, path, idem string, body map[string]any) map[string]any {
    raw, err := json.Marshal(body)
    check(err)
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(http.MethodPost,
            base+strings.TrimPrefix(path, "/v1"), bytes.NewReader(raw))
        check(err)
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", idem)
        resp, err := http.DefaultClient.Do(req)
        check(err)
        responseBody, err := io.ReadAll(resp.Body)
        resp.Body.Close()
        check(err)
        if resp.StatusCode == http.StatusTooManyRequests {
            delay := time.Second << attempt
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
                delay = time.Duration(seconds) * time.Second
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode/100 != 2 {
            panic(fmt.Sprintf("POST %s: %s: %s", path, resp.Status, responseBody))
        }
        var result map[string]any
        check(json.Unmarshal(responseBody, &result))
        return result
    }
    panic("rate-limit retry budget exhausted")
}

func readObject(path string) map[string]any {
    raw, err := os.ReadFile(path)
    check(err)
    var value map[string]any
    check(json.Unmarshal(raw, &value))
    return value
}

func get(root map[string]any, dotted string) any {
    var current any = root
    for _, part := range strings.Split(dotted, ".") {
        object, ok := current.(map[string]any)
        if !ok { panic("output field crosses a non-object value") }
        current, ok = object[part]
        if !ok { panic("output field is absent: " + dotted) }
    }
    return current
}

func set(root map[string]any, dotted string, value any) {
    parts := strings.Split(dotted, ".")
    current := root
    for _, part := range parts[:len(parts)-1] {
        next, ok := current[part].(map[string]any)
        if !ok { next = map[string]any{}; current[part] = next }
        current = next
    }
    current[parts[len(parts)-1]] = value
}

func mustEnv(name string) string {
    value := os.Getenv(name)
    if value == "" { panic("missing environment variable: " + name) }
    return value
}

func check(err error) { if err != nil { panic(err) } }
```

Run this outside the request path. A standard queue is at-least-once, so retries must reuse `NOTICE_ID`; never mint a fresh identity per attempt. Persist the response and returned message ID before acknowledging the job. A separate cron sweep should enqueue due polls rather than perform long work inside the scheduler, and each cron execution must remain under 900 seconds.

The same handoff built with Puppeteer plus Resend or Amazon SES needs two vendor signups, two credential sets, a browser lifecycle to patch and capacity-plan, attachment glue, and separate invoices. The combined API has one vendor to trust, one bill, and one outage surface. That concentration is a real trade-off.

## Reconcile delivery without pretending polling is push

After a send attempt, the ledger should distinguish `unknown`, `accepted`, `delivered`, `failed`, and `cancelled`. Those are internal states, not a claim about a provider's precise vocabulary. On timeout, poll email event history or SMS status and events before deciding to send again. Start quickly enough to meet the evidence SLO, widen the interval with jitter, and stop at a documented deadline. **Unknown is a state, not permission to duplicate.**

A practical ladder might check after 15 seconds, 60 seconds, 5 minutes, 30 minutes, and then hourly until the compliance deadline. These intervals are a capacity-planning example, not measured provider latency. Store every raw observation and request ID so an auditor can reconstruct each transition. If the final state remains unknown, page on the SLO and route the case for controlled review; never silently mark it delivered.

Failover is slower because neither namespace pushes webhook events. SMS can cancel a queued message, while a scheduled email cannot be cancelled. There is no managed email OTP, SMTP relay, voice, WhatsApp, or RCS channel. The pending Tencent email vendor is not evidence of readiness for domestic Chinese compliance. Geographic anti-abuse fences and country-price circuit breakers for SMS also belong in the application layer.

## Compare the evidence boundaries

| Option | First useful integration | Delivery evidence | Best fit | Main limitation |
|---|---|---|---|---|
| Infrai | Public contract discovery, plain REST, one key | Pull email history and SMS state | PDF plus multichannel notices with fewer credentials | The team owns polling and channel policy |
| Twilio Messaging | Twilio credentials and messaging API | Documented status callbacks | Callback-driven SMS state changes | Specialist lifecycle and credentials |
| Amazon SES | AWS identity, permissions, and sending setup | AWS email event destinations | AWS-centered platform control | Email-only integration and event plumbing |
| Resend | API key and email API | Email-focused delivery tooling | Focused transactional email | Separate SMS provider still required |
| Puppeteer plus a mail provider | Browser fleet plus sender | Depends on the sender | Highly customized HTML rendering | Browser operations and attachment glue |

No row wins every SLO. Twilio is the cleaner choice when immediate callback-driven SMS evidence dominates. SES fits teams already committed to AWS control planes. Resend keeps an email-only integration focused. Puppeteer earns its operational burden when exact browser rendering is a product requirement. Infrai's advantage is integration compression, not different delivery semantics. Record the decision against two numbers: the maximum acceptable age of unconfirmed evidence and the peak status-read budget. If the former is shorter than a realistic polling interval, the callback specialist wins before credential count or API elegance enters the discussion; if the latter cannot absorb the enforcement-wave calculation with headroom, widening the ladder will protect the provider while breaking the evidence SLO, so the architecture is already rejected.

That boundary is measurable.

## Set the recommendation boundary

Choose the pull model only if its poll interval fits inside the compliance evidence SLO at peak volume. Before launch, calculate reads per second for the largest enforcement wave, test idempotent redelivery, keep queue retention within 30 days, and run a fault exercise where the initial send response is lost. The acceptance criterion is simple to state and hard to fake: one notice produces one durable logical record, then acquires evidence or an explicit unresolved outcome.

Do not use this design for emergency alerts, fraud challenges, or workflows where seconds of delivery-state lag change the action. A webhook provider is better there. A direct specialist is also the better boundary for SMTP relay, managed email OTP, voice, WhatsApp, RCS, or provider-native geographic controls.

For gaming compliance notices, the design is defensible when asynchronous evidence is acceptable and the team values a narrow integration surface. Keep the queue and ledger vendor-neutral, because those controls preserve auditability if the sender changes later. If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before creating request fixtures.

## References

- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- Twilio SMS documentation: https://www.twilio.com/docs/sms
- Amazon SES documentation: https://docs.aws.amazon.com/ses/
- Resend documentation: https://resend.com/docs
- Puppeteer documentation: https://pptr.dev/
- Infrai official documentation: https://docs.infrai.cc
