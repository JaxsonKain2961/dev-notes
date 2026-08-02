# Is SMS OTP Enough for GDPR, PSD2, NIST, and 2FA Login Risk?

## TL;DR

SMS OTP is not enough as the only control for a high-risk login path in the EU or US. Use it as a recovery or step-up option, make a phishing-resistant factor the primary choice where the risk warrants it, and record the decision from a threat model rather than from the fact that a message arrived.

Start there.

I own a platform roadmap with the pager in mind, so my operational recommendation is direct: ship a factor that cannot be silently redirected by a carrier account takeover, keep SMS enrollment and recovery behind hard limits, and give product, security, and compliance teams the same evidence.

## Is SMS OTP enough for EU and US GDPR, PSD2, NIST, and 2FA login risk?

No, not by itself. GDPR does not prescribe a particular authenticator, but Article 32 requires security appropriate to the risk; a login design that assumes control of a phone number is permanent starts from a weak premise. PSD2 strong customer authentication requires two or more elements from knowledge, possession, and inherence, with those elements independent. That is a reason to assess the full authentication flow, not merely count the digits in a message. NIST SP 800-63B also calls out the risk of PSTN out-of-band authentication and says verifiers should consider SIM change or porting.

The signal that changes my runbook is not a failed delivery. It is a new device, a recent password reset, a changed phone number, an unfamiliar ASN, or a transaction beyond the account's normal pattern. An SMS code proves that somebody can currently receive messages at that number. It doesn't prove that the account holder initiated the login, and it does not bind a browser session to the person who enrolled the factor. A phishing page can collect a newly delivered code before the timer expires.

That distinction matters.

I've seen the adjacent operational failure firsthand: a cold-start path added 1,800 ms to p99 login latency under real traffic, although staging looked ordinary, and the retry storm made the authentication dashboard resemble credential stuffing. Different cause, same lesson: capacity planning and observability have to exist before an incident makes every signal ambiguous.

For regulated payment actions, map the exact action to PSD2 dynamic-linking and transaction requirements with counsel and the payment provider; a general login OTP is not a substitute. For ordinary consumer sign-in, the threshold may be lower, but the threat remains.

## Compare authenticators and message providers by the actual threat model

My comparison starts with attack resistance, recovery cost, and the SLO we can sustain. A WebAuthn passkey is usually the strongest modern consumer-login default because the authenticator checks the relying-party identifier, reducing the value of a convincing phishing page. TOTP apps avoid carrier routing and are portable, although their codes remain phishable in real time. SMS reaches nearly every phone and helps during a migration, yet SIM swaps, number recycling, and message interception keep it out of the primary position for valuable accounts.

| Option | Best fit | Principal trade-off | Operations question |
| --- | --- | --- | --- |
| WebAuthn passkey | Primary login on supported devices | Device-loss and recovery UX | Can enrollment and assertion meet the login SLO? |
| TOTP authenticator | Portable second factor | Real-time phishing | How are seeds and resets protected? |
| SMS through Twilio | Broad reach and recovery | Carrier takeover and phishing | Are number changes and delivery events monitored? |
| SMS through Amazon SNS | An AWS-operated communications stack | Same phone-number threat model | Are IAM and message-spend limits isolated? |
| Email recovery through Resend | Teams that need transactional recovery email | Inbox takeover remains possible | Are domain authentication and reset events audited? |

Twilio and Amazon SNS are delivery transports, not distinct security factors, and Resend changes email delivery rather than the assurance of an email-based recovery link. I would choose among transports by destination coverage, support, sender registration, and the telemetry the on-call team can use. Your mileage may vary by country.

No transport fixes that.

The catch is that a passkey is not suitable when the application cannot provide a credible device-loss recovery path or the user population has constrained hardware. Use TOTP as the normal second factor in that case, then make recovery a tightly reviewed process instead of pretending that SMS changes its security properties. If email participates in recovery, publish and verify SPF as described in RFC 7208; an inbox takeover can undo an otherwise careful factor choice.

## Implement a safe TOTP verification path

This dependency-free Go program verifies a six-digit, SHA-1 TOTP code using the RFC 6238 30-second time step. It compiles with the Go standard library, which makes it useful for testing the verification rule before it sits behind a login service. In production, store each enrollment secret in a managed secret store, encrypt it with a separately controlled key, rate-limit attempts per account and IP, and never log the secret or submitted code.

```go
package main

import (
	"crypto/hmac"
	"crypto/sha1"
	"encoding/base32"
	"encoding/binary"
	"fmt"
	"time"
)

func validTOTP(secret, code string, now time.Time) bool {
	key, err := base32.StdEncoding.WithPadding(base32.NoPadding).DecodeString(secret)
	if err != nil || len(code) != 6 {
		return false
	}
	buf := make([]byte, 8)
	binary.BigEndian.PutUint64(buf, uint64(now.Unix()/30))
	h := hmac.New(sha1.New, key)
	h.Write(buf)
	sum := h.Sum(nil)
	offset := sum[len(sum)-1] & 15
	v := (uint32(sum[offset])&127)<<24 | uint32(sum[offset+1])<<16 | uint32(sum[offset+2])<<8 | uint32(sum[offset+3])
	return fmt.Sprintf("%06d", v%1000000) == code
}

func main() {
	fmt.Println(validTOTP("JBSWY3DPEHPK3PXP", "000000", time.Now()))
}
```

Accept only a narrow clock window if one is needed, record the reason for every factor reset, and require a recent stronger factor before changing a factor. Don't make the login handler synchronously wait on optional analytics or a profile lookup — a clean authentication SLO matters more than a pretty event stream. I'm not sure why recovery changes still bypass the highest-risk policy in some designs, but that is where a sound login system often loses its margin.

## Verify, monitor, and roll back without lowering assurance

Before rollout, test passkey and TOTP enrollment, assertion, recovery, and factor removal on actual browsers and device classes, then run synthetic probes against the full login path from important regions. Track success rate, p50 and p99 latency, challenge abandonment, recovery volume, SMS delivery failures, number-change attempts, and time from suspicious event to account protection. Alert on a sustained baseline deviation with a runbook that separates a carrier delivery issue from an elevated takeover signal.

Measure the recovery path separately: it usually has a different dependency chain, lower traffic, less mature dashboards, and a higher concentration of deliberate abuse than the normal assertion flow, so a single aggregate login-success metric can look healthy while the control that permits account takeover has become slow, permissive, or impossible for a legitimate user to complete. Give it an explicit error budget, test carrier and inbox alternatives, and ensure that a manual-review queue has a staffed owner before a large incident turns it from an exception process into the primary authentication system.

Deploy behind a cohort flag and retain the prior, stronger completed session as the rollback safety net. A rollback should stop new enrollment or return a user interface to an already enrolled factor; it must not quietly lower every account to password plus SMS. For payment flows, involve security, legal, and acquiring teams before changing the challenge policy, because transaction binding and evidence matter alongside a green login metric.

The capacity question is concrete: can the service preserve its authentication SLO while a large share of users enrolls a new factor after a security notice? Prebuild indexes for account and factor state, load-test the rate limiter, and verify that provider callbacks cannot become a replay channel. Keep an audit trail with request IDs and actor context, while minimizing personal data and retention so the audit system does not create the next privacy problem.

## References

- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://www.eba.europa.eu/regulation-and-policy/payment-services-and-electronic-money/regulatory-technical-standards-on-strong-customer-authentication-and-secure-communication-under-psd2
- https://www.w3.org/TR/webauthn-3/
- https://www.rfc-editor.org/rfc/rfc6238
- https://datatracker.ietf.org/doc/html/rfc7208
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
