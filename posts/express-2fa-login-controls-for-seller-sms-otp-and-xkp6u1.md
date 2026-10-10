# Express 2FA Login Controls for Seller SMS OTP and Email Fallback Templates

Keep the OTP lifecycle in the authentication service, but let the marketplace team own the SMS and email templates as versioned code. The deciding constraint is whether an on-call engineer can prove that the code, destination, locale, order context, and fallback policy all belonged to one login attempt without placing the OTP itself in logs.

**TL;DR:** for a seller opening a new-order notification, issue one short-lived challenge, render two channel-specific messages from the same reviewed template release, and treat email as an explicit state transition rather than a second code sent on a timer. The Express application should create and verify challenges through a narrow internal interface. Delivery workers should receive an opaque challenge ID and authorized contact references, while the verifier stores only a keyed digest of the code.

This division makes rollback boring. A bad wording release can be reverted independently of the authentication state machine, and a transport problem can be contained without changing what counts as a valid login.

## How should Express 2FA login coordinate SMS OTP and email fallback?

The new-order message is a customer-support artifact, a security prompt, and a route into a seller account. A centralized communications group may understand carrier behavior and email delivery, while the marketplace team understands whether an order reference is safe context, which locales are supported, and which page should follow successful verification. Giving either side the entire feature creates an ownership gap at review time.

The split is clear: the authentication service owns code generation, expiration, attempt accounting, replay prevention, and verification. The marketplace team owns words, locale selection, approved order fields, and template releases. The transport adapter owns encoding, provider response normalization, retry classification, and delivery telemetry. No template can change security state. No transport callback can mark a challenge verified.

That boundary is non-negotiable.

Keep three signals separate: challenge outcome, submission outcome, and transport outcome. An accepted send request is not proof that a person received a message, and a delivery event is not proof that the recipient entered the code. Combining them produces reassuring dashboards that cannot answer the incident question: did sellers fail to authenticate, or did one channel merely report slowly?

A capacity plan should start with attempts, not accounts. For each new-order notification, estimate the fraction that triggers sign-in, the allowed resend and verification policy, and the proportion eligible for email fallback. Peak load is driven by challenges and sends per challenge. Do not bury retries inside a provider adapter, because invisible retries consume transport capacity while the application believes there was one send.

| Ownership choice | On-call effect | Lock-in surface | Template rollback |
|---|---|---|---|
| Transport-managed templates | Fewer rendering components, but changes cross a control-plane boundary | Template identifiers and substitution rules | Authentication may work while prior wording remains deployed |
| Team-owned templates with managed delivery | Rendering and delivery have separate alerts | Transport request and event schemas | Revert wording without rotating challenges |
| Team-owned templates and self-operated delivery | The team owns queues, reputation, routing, and rendering | Lowest external template coupling | Local rollback, larger operational blast radius |

I would choose the middle row unless control of the delivery plane is itself a requirement. This is a buy-versus-build decision about on-call load, not a claim that managed delivery is inherently safer. The exit plan is the normalized transport interface and an event schema the application owns.

The limitation is operational maturity. Team-owned rendering with managed delivery is a poor fit when nobody can own locale review, template compatibility, or an after-hours rollback; in that case, a transport-managed template system narrows the team's responsibility, at the cost of coupling releases to that control plane. Self-operated delivery is also a poor fit when the team cannot staff queue operations and sender-reputation work. Those are real trade-offs. Owning a template repository creates review and deployment duties, and an abstraction around delivery still needs integration tests for every adapter. The table is a decision record, not a ranking.

## Model fallback as a challenge transition

Fallback is a state change.

A fallback button should not mint an unrelated email code. It should transition the same challenge from `sms_pending` or `sms_unavailable` to `email_pending`, invalidate prior send intents, and retain one verification policy. Otherwise, two valid secrets can race, support cannot explain which message won, and resend controls fragment by channel.

Do not infer failure from silence after an arbitrary client-side countdown. A seller may request email because the phone is unavailable, because the SMS submission was rejected, or because the message has not arrived. Those are different reasons, but the security rule can remain the same: authorize the channel change on the server, record a reason category, and produce one new send intent tied to the existing challenge.

The following Go sketch shows the boundary. Exact expiry, attempt limits, and resend intervals are policy inputs; hard-coding universal values would turn an example into an undocumented security claim.

```go
package auth

import (
    "context"
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "errors"
    "time"
)

type Channel string
const ( SMS Channel = "sms"; Email Channel = "email" )

type Challenge struct {
    ID, OrderReference, CodeDigest, TemplateRelease string
    ExpiresAt time.Time
    ActiveChannel Channel
}
type SendIntent struct {
    ChallengeID, DestinationRef, Template, Locale, IdempotencyKey string
    Channel Channel
    Fields map[string]string
}
type Store interface {
    TransitionChannel(context.Context, string, Channel, Channel) (Challenge, error)
    ConsumeAttempt(context.Context, string, string, time.Time) (bool, error)
}
type Queue interface { Publish(context.Context, SendIntent) error }
type Service struct { store Store; queue Queue; macKey []byte; now func() time.Time }

func (s *Service) FallbackToEmail(ctx context.Context, id, destinationRef, locale string) error {
    c, err := s.store.TransitionChannel(ctx, id, SMS, Email)
    if err != nil { return err }
    return s.queue.Publish(ctx, SendIntent{
        ChallengeID: c.ID, Channel: Email, DestinationRef: destinationRef,
        Template: "seller-order-sign-in", Locale: locale,
        Fields: map[string]string{"order_reference": c.OrderReference},
        IdempotencyKey: c.ID + ":email:" + c.TemplateRelease,
    })
}

func (s *Service) Verify(ctx context.Context, id, code string) error {
    ok, err := s.store.ConsumeAttempt(ctx, id, s.digest(id, code), s.now())
    if err != nil { return err }
    if !ok { return errors.New("invalid or expired challenge") }
    return nil
}
func (s *Service) digest(id, code string) string {
    mac := hmac.New(sha256.New, s.macKey)
    mac.Write([]byte(id)); mac.Write([]byte{0}); mac.Write([]byte(code))
    return hex.EncodeToString(mac.Sum(nil))
}
```

The Express route is deliberately thin: validate the request, call `FallbackToEmail`, and return a generic result that does not disclose whether an address exists. The JavaScript layer should not render messages, compare codes, or translate provider errors into authentication success. This keeps the language boundary from becoming a second implementation of the protocol.

`DestinationRef` is a lookup key, not a phone number or email address copied through every queue. Resolve it at the last responsible component under the same authorization context. The order reference is display context only; possession of it must grant nothing.

## Make releases observable without logging secrets

Attach a stable template name, immutable release identifier, locale, channel, and challenge ID to the send intent. Record rendered byte length and the encoding category for SMS, but never the code or complete rendered body. SMS segmentation depends on encoding: the cited reference documents 160 GSM-7 characters for a single segment and 70 UCS-2 characters, with lower per-segment limits for concatenated messages. A translated template can therefore change message count without adding many visible characters.

Set an SLO on the user outcome, then use transport metrics as diagnostics. The primary indicator is the proportion of valid, non-expired challenges completed within the product's chosen window. Supporting views should split challenge creation, queue delay, rendering rejection, submission outcome, channel transition, and verification result by template release and locale.

Page on sustained inability to create or verify challenges, because those are direct login failures. Route a single-locale rendering rejection to the owning team as a release issue. Treat a rise in email fallback as a diagnostic signal first; it may indicate SMS trouble, but it may also reflect user choice or a product-flow change.

The email adapter needs the same discipline. Amazon SES documentation distinguishes its API from its SMTP interface and describes sandbox restrictions, sending quotas, and identity verification. Those delivery-plane constraints belong in readiness checks; they should not leak into the challenge state machine or template model. Another transport may expose different controls, which is why the adapter boundary exists.

## Verify, release, and roll back

Start with deterministic rendering tests. For every supported locale, render SMS and email from fixed fields, reject missing substitutions, and snapshot the result with a non-secret OTP fixture. Assert that HTML escaping occurs in the renderer and that the plain-text email remains usable. Measure SMS encoding and segments using documented rules rather than a character-count guess.

Then test state transitions under concurrency. Two simultaneous fallback requests must yield one active email intent. A verify request racing expiry must have one authoritative result from storage. A consumed challenge must remain consumed after a resend request. These tests belong at the repository boundary, where compare-and-set behavior and transactions are real.

Before broad release, use non-production destinations and a template release that cannot address real sellers. Exercise new-order notification, SMS request, manual email fallback, resend, wrong code, expired code, and successful verification. Confirm that dashboards can follow one opaque challenge without revealing the destination or code.

A release gate can be compact:

1. Template snapshots pass for every declared locale and channel.
2. The rendered SMS is classified for encoding and segment count.
3. Duplicate fallback and resend requests preserve idempotency.
4. Verification logs contain result categories, never submitted codes.
5. The previous template release remains deployable.

Five checks. No ceremony.

Rollback the template first when the failure is content, rendering, localization, or unexpected SMS segmentation. Pin new send intents to the previous immutable release and leave existing challenge expiry and attempt accounting alone. A challenge already issued should not become longer-lived because the message around it changed.

If one delivery channel is impaired, stop creating new intents for that channel and allow the server-authorized transition to the other channel. Do not automatically send both: that doubles sensitive messages, complicates capacity forecasts, and makes the active path ambiguous. Prevent stale work from sending after its challenge can no longer be verified.

Compare challenge completion and rendering failures by template release, channel, and locale while holding the authentication policy version constant. If completion recovers after rollback, the evidence points to the release. If creation or verification remains impaired across releases and channels, inspect the shared challenge path.

This design leaves the Express application small, gives the marketplace team control over seller-facing language, and keeps security state independent from delivery vendors. It also answers who changed the message, which release was sent, why fallback occurred, and whether the challenge succeeded, without turning an OTP into telemetry.

## References

- Amazon Simple Email Service documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
