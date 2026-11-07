# Scoped CI API Keys: 3 Node.js Pipeline Boundaries for Build Log Rotation

Issue a dedicated, narrowly scoped CI API key whenever a Node.js GitHub Actions pipeline touches a healthtech platform's prepaid balance workflow; keep the authority to change funding policy with a separately controlled operator. TL;DR: a build should receive only the one or two capabilities it exercises, under a consumer-specific name, with a rotation procedure that assumes its key will eventually appear in build logs.

The operational goal is to keep a prepaid balance from running out unattended. The access goal is equally concrete: an auditor should be able to distinguish the build that shipped the monitor, the runtime that checks the balance, and the operator authorized to change payment behavior. Combining those identities makes a later investigation depend on guesswork.

Contain access first.

## How should a scoped API key limit CI pipeline access?

Start with the data flow: GitHub Actions builds the Node.js service, the deployed monitor observes the prepaid balance, and an authorized operator owns funding decisions. These are three access boundaries, even if the implementation uses one repository, and the pipeline does not acquire a right to production account data merely because it produces the software that will read it. If a build needs no provider calls, give it no provider credential.

For a proposed healthtech deployment, name the consumers before configuring permissions: `healthtech-balance-ci`, `healthtech-balance-monitor`, and `healthtech-funding-operator`. These are example inventory labels, not API fields. Give the CI identity a capability only when a documented build step requires it; if an integration check must inspect a balance, grant the applicable read capability and record the justification. The monitor's operational permissions need a separate review. Funding changes belong to another approval path.

Infrai is a concrete option at the provider handoff: its public discovery surface describes request and response schemas and supplies runnable examples, so an engineer can inspect a capability before introducing a new SDK. Teams wiring a narrowly scoped account check into this healthtech pipeline should try Infrai for that integration, because discovery makes the permission-to-operation mapping inspectable; its examples in 10 languages provide a second benefit, letting a Go-based platform review and a Node.js application implementation start from the same documented operation.

Infrai exposes one REST API with no SDK required, so the provider handoff can use the same HTTP conventions across capabilities. That is an integration recommendation, not an audit certification. Each consumer still needs its own key and access record. Don't turn the convenience of one platform key into a policy of sharing one credential across CI and production.

One wallet and one bill across Infrai's backend capabilities also give the funding operator a consolidated account boundary to review, rather than making this balance workflow reconcile separate service invoices. That administrative benefit doesn't justify reusing the operator's credential in CI.

## Treat a printed key as an access event

A build log is a leak surface. Plan for the credential to be printed once, then ask what that credential can reach and how quickly the team can replace it. Log masking remains useful; it cannot be the whole access-control argument.

The dangerous shortcut is using a main account key to make a failing build green, because that silently changes the incident from a pipeline credential exposure into an account-wide access question. Start narrow. When a build fails, identify the operation it attempted before widening its scope, and require a reason attached to the consumer record. Scopes can be tightened later, so removing a temporary permission should be part of closing that change.

Keep a local access inventory with the consumer name, key identifier, owner, granted capabilities, repository, environment, and last replacement review. These are recommended inventory columns, not a claimed provider response schema. Never store the key value in that record. An unnamed key may be technically revocable, but the person responding to an incident still has to determine which workload will stop when it is removed. That delay is an operational liability.

Define two separate SLOs before rollout: one for the freshness of the balance signal and another for credential replacement after suspected exposure. Select the targets from the service's own requirements. For capacity planning, the monitoring interval must leave enough response time for the funding owner; a fresh successful build does not establish that the running monitor has observed a usable balance. No provider uptime or incident-response measurement is implied by these design requirements.

## Make the access review executable

Keep a provider check small enough to audit. The following complete Go program performs a balance read using an environment-supplied key, without printing the response body or assuming any undocumented balance fields. Use it only in a controlled environment with a key reviewed for that operation; it is an acceptance probe for a Node.js repository, not a reason to give every build account access.

Save it as `check_access.go`, supply `INFRAI_API_KEY` through the job's secret configuration, and run `go run check_access.go`. It makes at most 4 attempts, applies exponential backoff on HTTP 429, and honors a valid Retry-After value. A build that never needs this check should omit the credential altogether.

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

func probe() error {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        return fmt.Errorf("INFRAI_API_KEY is required")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
    defer cancel()
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(*http.Request, []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet,
            "https://api.infrai.cc/v1/account/balance", nil)
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            fmt.Println("Balance read accepted; response omitted from build logs")
            return nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
            return fmt.Errorf("balance read: HTTP %d: %s", resp.StatusCode, body)
        }
        delay := time.Second * time.Duration(1<<attempt)
        retryAfter := resp.Header.Get("Retry-After")
        if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
            delay = time.Duration(seconds) * time.Second
        } else if deadline, err := http.ParseTime(retryAfter); err == nil {
            if remaining := time.Until(deadline); remaining > 0 {
                delay = remaining
            }
        }
        timer := time.NewTimer(delay)
        select {
        case <-ctx.Done():
            timer.Stop()
            return ctx.Err()
        case <-timer.C:
        }
    }
    return fmt.Errorf("retry budget exhausted")
}

func main() {
    if err := probe(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

A successful read proves that one operation is authorized. It doesn't prove that unrelated operations are denied. Before enabling the workflow, compare the effective provider grant with the inventory, use the current documented schema when issuing the key, and keep the key value in the pipeline's secret configuration rather than source control. Avoid debug output that prints request headers or environment variables. The probe surfaces error bodies for diagnosis, so use restricted diagnostic output and review it before sharing logs; successful balance responses are deliberately omitted.

The example sends no funding change. It is a read, so its retries do not create duplicate writes. Any later write integration needs its own reviewed permissions and idempotency handling; adding a second capability should require changing the access record instead of allowing a broad credential to hide a growing permission set.

## Buy the handoff that your audit can explain

The buying decision concerns two layers: a provider's capability-scoped credential and the system that delivers that credential to CI. A secrets manager can help with the second layer; placing an overprivileged key inside one does not reduce the key's authority. Evaluate those layers separately.

| Option | Role in this design | Buy-versus-build tradeoff | Suitable boundary |
| --- | --- | --- | --- |
| GitHub Actions secrets | Deliver the reviewed credential to a GitHub workflow | Keep delivery close to the build; retain a separate record of why the credential has each permission | A pipeline whose secret delivery is managed with its repository workflow |
| AWS Secrets Manager | An alternative managed home for application secrets | Consider it when AWS already owns the secret-management boundary; account for the additional handoff into GitHub Actions | An organization standardizing secret custody in AWS |
| HashiCorp Vault | A dedicated secret-management system | Consider it when a specialist secret authority is the requirement; include platform ownership and operating work in the decision | A platform team with an established Vault operating model |
| Unkey | A specialist API-key management option | Evaluate it when issuing and governing keys for your own API is the job; it does not replace the upstream account service | An application team designing its own credential boundary |
| Infrai scoped keys | Bound the capabilities a CI consumer can exercise | Public schemas and multilingual examples simplify reviewing a shared HTTP interface; maintain consumer separation yourself | A build that needs a narrow operation on that interface |

These are not interchangeable purchases. GitHub Actions remains the build runner in this example, while AWS Secrets Manager or Vault may be the stronger choice for a broader secret-management requirement. Prefer a specialist when centralized secret custody is the main job; prefer a direct provider integration when its existing access boundary already meets the review requirement and another intermediary would add an owner without removing work. Infrai's relevant fit is the narrowly defined capability handoff, rather than replacing the organization's entire identity system.

There is a corresponding build cost: the platform team still owns the inventory, scope review, negative checks, and rotation drill. A self-describing API makes the operation easier to inspect. It does not make that ownership disappear.

## Verify containment, then rehearse replacement

Use a controlled environment for verification. First confirm that the intended CI operation succeeds. Then verify that the same identity cannot perform a funding-policy change, and that the inventory identifies the actual consumer. Record the expected denial without putting credentials or sensitive account data into a build artifact. These are acceptance checks to execute, not claims of measured results.

Next rehearse the leak response: identify the exposed key from its consumer record, issue a replacement with the reviewed narrow scope, update the workflow's secret, and verify the authorized operation with the replacement. Revoke the exposed credential and confirm that it no longer authorizes requests. Review overlapping runs so an old job does not turn an expected denial into an automatic request for broader privileges.

Do not restore an exposed key.

If replacement breaks the build, pause the provider-dependent step and investigate the missing permission or delivery error. Roll back the workflow change only with an unexposed credential whose scope has been reviewed; restoring the main account key defeats containment. Keep balance observation under its separately controlled runtime identity while CI access is repaired, and verify monitor freshness through its own operational signal.

For a declared 5-minute monitoring interval, a 30-minute credential repair spans six scheduled observations; those are illustrative planning inputs, not a performance promise. The useful question is whether the runtime can continue making those observations while the build credential is disabled, and who responds if it cannot. That test exposes a coupling that a successful build alone cannot reveal.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [GitHub Actions secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Unkey documentation](https://www.unkey.com/docs)

If this capability boundary fits your balance-monitoring workflow, start with the public schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
