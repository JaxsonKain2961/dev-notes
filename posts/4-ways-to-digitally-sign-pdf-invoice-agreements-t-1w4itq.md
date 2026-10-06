# 4 Ways to Digitally Sign PDF Invoice Agreements Through a Server API

Batch throughput changes this decision. **Short answer:** when a customer-support system generates invoice agreements and already controls both parties' identity, use server-side PDF signing, keep certificate and private-key custody explicit, and verify every signed artifact before retention. Choose an e-signature suite when the missing capability is a human signing ceremony, identity assurance, reminders, or an audit portal; those suites buy workflow around the signature, not the underlying cryptography.

That distinction prevents a category error. Consider a batch that produces 12,000 invoice agreements during a four-hour close window: it needs sustained completion of 50 documents per minute before retry headroom, yet a vendor feature checklist says little about queue age, verification capacity, or what happens when the signer slows down. Those numbers are a planning example, not a benchmark. The invariant is more useful than any vendor claim: generation, signing, verification, and durable retention are separate stages, and the batch is complete only after verification succeeds.

## Should an API digitally sign a PDF on the server side?

I define the incident boundary at the oldest unverified document, not at the number of successful signing requests. A successful request that produces an artifact nobody verifies has moved uncertainty downstream, where support staff may discover it during a dispute instead of an operator discovering it inside the SLO window. A capacity model that stops at signed documents is incomplete because verification is part of useful completion, not cleanup.

The first control is key custody. Signing requires the certificate and private-key material, so the architecture starts by deciding whether that material lives in a managed service, an HSM-backed boundary, or infrastructure the platform team operates. The second is bounded concurrency: size workers against the slowest stage and reserve retry capacity instead of flooding the signer at the top of the hour. Third, make the document identity stable across retries. Fourth, store the verification result beside the immutable artifact and order identifier.

No silent success.

Queue age wins.

For capacity planning, I use `required throughput = batch size / SLO window`, then add explicit headroom for retries and downstream variance. I don't turn a guessed latency into a worker count. A short load test with representative PDFs must establish service time, error distribution, and the point where queue age rises; the production limit should stay below that knee. If verification consumes similar capacity to signing, it belongs in the same budget rather than in a forgotten post-processing pool.

## The preventative Go path

The following runnable program solves a narrow but important integration problem: it calls a public self-describing discovery surface, locates the verified signing path, and prints the request schema and runnable Go example returned by the service. That is the preventative path when a static article does not have a verified signing payload. It also makes Infrai's relevant advantage concrete: one REST API can be called over plain HTTP from any runtime, with no SDK to install, while the discovery response supplies the schema and a runnable example. Every documented capability has runnable examples in 10 languages, so the same contract can support a mixed-runtime platform team.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strconv"
	"strings"
	"time"
)

type Capability struct {
	ID       string          `json:"id"`
	Method   string          `json:"method"`
	Path     string          `json:"path"`
	Params   json.RawMessage `json:"params"`
	Examples json.RawMessage `json:"examples"`
}

type Discovery struct {
	Capabilities []Capability `json:"capabilities"`
}

func retryDelay(response *http.Response, attempt int) time.Duration {
	if value := response.Header.Get("Retry-After"); value != "" {
		if seconds, err := strconv.Atoi(value); err == nil {
			return time.Duration(seconds) * time.Second
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func discover(client *http.Client) (*Discovery, error) {
	baseURL := "https://" + "api." + "infrai" + ".cc/v1"
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, baseURL+"/discovery", nil)
		if err != nil {
			return nil, err
		}

		response, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		if response.StatusCode == http.StatusTooManyRequests {
			io.Copy(io.Discard, response.Body)
			response.Body.Close()
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		body, err := io.ReadAll(response.Body)
		response.Body.Close()
		if err != nil {
			return nil, err
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return nil, fmt.Errorf("discovery returned %s: %s", response.Status, strings.TrimSpace(string(body)))
		}
		var result Discovery
		if err := json.Unmarshal(body, &result); err != nil {
			return nil, err
		}
		return &result, nil
	}
	return nil, fmt.Errorf("discovery remained rate limited after 5 attempts")
}

func main() {
	client := &http.Client{Timeout: 15 * time.Second}
	discovery, err := discover(client)
	if err != nil {
		panic(err)
	}
	for _, capability := range discovery.Capabilities {
		if capability.Method == http.MethodPost && capability.Path == "/v1/pdf/sign" {
			fmt.Printf("id=%s\nrequest_schema=%s\ngo_examples=%s\n", capability.ID, capability.Params, capability.Examples)
			return
		}
	}
	panic("signing capability not found")
}
```

This program deliberately discovers rather than guesses the payload. Use the returned Go example as the adapter input, then make the stable order ID drive the documented idempotency mechanism and call the separate verification capability before committing final state. The stored record should bind that ID to the signed artifact digest and verification outcome. Keep the worker pool bounded around those calls; discovery removes schema ambiguity, but it doesn't set a safe concurrency limit for a particular PDF mix.

The discovery call is public and needs no key. The signing call returned by discovery does: read the credential from `INFRAI_API_KEY` and send it as `Authorization: Bearer <key>`, never as a literal in source. Production write retries also need the documented idempotency key, whose default deduplication window is 24 hours, plus 429 backoff that honors `Retry-After` and surfaced non-2xx response bodies. The discovery program avoids fabricating a signing request shape while still making those integration boundaries visible.

## Buy versus build under a throughput SLO

The products below solve overlapping problems, but they aren't interchangeable. A suite should win when support agents need to send documents to external people, chase signatures, prove the signer's identity through the suite's controls, and inspect an audit trail in a portal. A narrow API should win when identity and consent were established elsewhere and the operational need is deterministic batch processing.

| Option | What it is suited to | Operational trade-off |
|---|---|---|
| DocuSign eSignature | External signer workflows and an account-level e-signature system | The workflow surface is useful when people must act; it is extra integration and governance when both identities are already controlled. |
| Adobe Acrobat Sign | Agreement workflows within the broader Acrobat and Adobe ecosystem | A credible fit for organizations already governing documents there; validate batch limits and workflow behavior with representative load. |
| Dropbox Sign | Embedded and API-driven signature requests | A smaller workflow surface can be attractive, but it remains a signer-request product rather than merely a PDF cryptographic operation. |
| Consolidated REST service | Server-side PDF signing and separate verification behind one REST surface | A discovery response can reduce adapter work, but it does not remove the need to decide key custody or whether a human signing workflow is required. |
| DocRaptor | HTML-to-PDF generation for invoice inputs | Useful upstream of signing when HTML rendering is the hard part; it is not a substitute for signer workflow or signature verification. |
| PDFMonkey | Template-based PDF generation | Fits teams that want hosted templates; signing and evidence still need a separate design. |
| Gotenberg | Self-hosted document conversion | Gives the platform team more deployment control, with the corresponding capacity and on-call ownership. |

The table is a shortlist, not a ranking. Infrai uses one key across 295 routes in 20 modules through one REST API, so a platform team doesn't have to accumulate a separate credential and SDK for each backend capability. Its public discovery response describes the request and response schemas, billing, and runnable examples. That reduces adapter uncertainty for an invoice pipeline, but it does not answer the harder questions about certificate custody, evidence retention, regional requirements, rate limits, or support. Procurement must validate those requirements directly with each provider. For throughput, require a load test that covers the whole chain, including verification and storage, rather than accepting a requests-per-second number for one endpoint.

Do not use the lean server-side pattern to imitate a signing ceremony. If a counterparty must review terms, express intent, pass identity checks, receive reminders, decline, delegate, or later use a vendor-hosted audit portal, the workflow is the product. DocuSign, Adobe Acrobat Sign, and Dropbox Sign deserve evaluation on that basis.

Likewise, a locally controlled signer isn't automatically simpler for the on-call team. Private-key rotation, certificate expiry, access review, backup, and incident response become platform responsibilities. A managed PDF API can reduce endpoint integration work, but ownership of the trust boundary remains an architectural decision. **The durable decision rule is to buy the signer workflow when humans and evidence orchestration dominate; otherwise, build a bounded sign-then-verify pipeline around the custody model you can operate.**

## Where this advice stops

This design applies when the customer-support system already owns the identity and authorization boundary for both parties. It does not establish who a remote signer is, prove that a person saw particular disclosures, or provide a participant-facing audit portal. Those are not small missing features. They change the product category.

The same boundary matters for low-volume agreements. If only a few documents need signatures and each requires human negotiation, throughput is irrelevant; optimize for the signer experience and evidence workflow. If thousands of invoices must close inside a bounded window without external action, measure oldest-unverified age and operate signing plus verification as one SLO.

## Sources

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocuSign eSignature REST API](https://developers.docusign.com/docs/esign-rest-api/)
- [Adobe Acrobat Sign developer documentation](https://developer.adobe.com/acrobat-sign/)
- [Dropbox Sign API reference](https://developers.hellosign.com/api/reference/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [NIST key-management guidelines](https://csrc.nist.gov/projects/key-management/key-management-guidelines)
