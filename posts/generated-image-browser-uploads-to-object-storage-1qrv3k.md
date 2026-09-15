# Generated Image Browser Uploads to Object Storage: CORS and Presigned PUT Troubleshooting

Short answer: for a marketplace receipt that must survive an audit and then be deleted on schedule, make the browser upload a small, observable state machine: verify the presigned PUT and CORS preflight separately, record the object key before acknowledging the receipt, and let a retention worker own deletion.

The browser error is often the least useful part of this incident. A failed preflight, an expired signature, a mismatched `Content-Type`, and a successful object write followed by a lost database row can all look like “the upload failed” from a checkout screen. Retention makes the distinction more important: an object that cannot be found may be an upload failure, a deliberate deletion, or an audit record that never pointed at the original file.

## Start with the receipt ledger, not the browser console

The bounded failure pattern is straightforward. A marketplace checkout creates a receipt in the application, the browser receives a time-limited PUT URL, and the receipt file is sent to object storage. The browser reports a CORS error. An engineer changes the signature code. The next test still fails, because the request never reached storage: the preflight was rejected before the signed write could be evaluated.

I don't collapse the gates in the incident record. Gate one is the browser's origin negotiation: the origin, requested method, and requested headers must be accepted by the storage endpoint. Gate two is the signed request: the method, object key, expiry, and any signed headers must match the request that actually leaves the browser. A `403` from storage and a browser-side CORS message are different evidence.

The same distinction helps with generated images, too, but receipts add a hard business invariant: the original file must remain associated with an immutable receipt identifier until its retention deadline. The object key should be derived from that identifier, not from a mutable filename. Store the key, content type, byte length, and upload state in the receipt record; acknowledge the receipt only after the write is confirmed. That record is the control plane for the file, and losing it is a retention failure even when storage accepted every byte.

Keep the key stable.

## What does a failed browser direct upload prove about CORS and a presigned PUT?

Start with a reproduction that does not involve the full checkout UI. Capture the browser origin, the requested method, the requested headers, and the URL expiry without logging the signed URL itself. Then test the preflight and the signed PUT as separate operations. A browser may send an `OPTIONS` request before the `PUT`; a server-side client normally does not have that browser-origin step.

The most common mismatches are mechanical:

- The URL was signed for `PUT`, but the client sends `POST`.
- The signature covered `Content-Type: application/pdf`, while the browser sends a different value or omits it.
- The storage CORS policy does not allow the checkout origin or the requested header.
- The URL expired while a user kept the payment page open.
- A client adds an authorization header intended for the application API to the signed storage request.

Do not retry all of these. Retrying a preflight cannot repair a policy mismatch, and retrying an expired URL only repeats a known failure. Reissue the URL after the application confirms that the receipt is still in an uploadable state. For a `429`, use bounded backoff; for a `403`, preserve the request metadata and investigate the signed contract.

Here is a deliberately small Go probe for a backend-owned test. It checks the signed PUT response without pretending to reproduce the browser's CORS enforcement. That limitation matters: a successful server-side PUT proves the storage write can work, not that the browser's preflight will pass.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
)

func main() {
	signedURL := os.Getenv("RECEIPT_PUT_URL")
	if signedURL == "" {
		panic("set RECEIPT_PUT_URL")
	}

	pdf, err := os.ReadFile("receipt.pdf")
	if err != nil {
		panic(err)
	}

	req, err := http.NewRequest(http.MethodPut, signedURL, bytes.NewReader(pdf))
	if err != nil {
		panic(err)
	}
	// Keep this request identical to the headers covered by signing.
	req.Header.Set("Content-Type", "application/pdf")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)

	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		panic(fmt.Sprintf("receipt PUT returned %d: %s", resp.StatusCode, body))
	}
	fmt.Println("storage accepted the receipt bytes")
}
```

The test output belongs beside the receipt state transition, with a request ID and timestamp. It should not include credentials or a reusable signed URL. If this probe succeeds while the browser fails, move the investigation to the preflight response and the deployed origin configuration.

## How should a retention clock drive deletion of an auditable receipt?

Retention is a policy, not a bucket setting that can be left implicit. Define the retention start event, the legal or business duration, the timezone used for the deadline, and the behavior after deletion. A useful record has at least `receipt_id`, `object_key`, `region`, `uploaded_at`, `retain_until`, `deleted_at`, and a state such as `pending`, `available`, or `deleted`.

The deletion worker should be idempotent. It can claim due rows, issue a delete, and record the result; a second run must not turn an already deleted object into a false upload failure. Keep the receipt metadata after the original file is removed when an audit trail requires proof that deletion occurred. Do not keep a hidden copy in a retry queue, application log, temporary directory, or backup without giving that copy its own retention rule.

There is a sharp trade-off here. A short retention period reduces storage exposure and cleanup work, but it narrows the window in which an auditor can inspect the original. A long period preserves evidence, but raises storage cost, access-control exposure, and deletion obligations. AWS documents storage pricing as a combination of storage and request or retrieval-related charges, so capacity planning should model both the receipt volume and the number of reads and deletion attempts rather than only the average file size.

| Design choice | Good fit | Cost or risk to accept |
| --- | --- | --- |
| Browser direct PUT | Large client files or a backend that must avoid carrying file bytes | CORS policy, preflight diagnostics, and signed-header discipline become part of the product path |
| Backend upload | Small receipts and teams that want one authorization boundary | Application bandwidth, buffering limits, and a larger service SLO surface |
| Managed object storage | A team that wants durable object primitives without operating disks | Provider policy, region availability, pricing, and exit planning remain external dependencies |
| Self-hosted object storage | A team that needs control over deployment and data placement | The team owns upgrades, replication, recovery tests, and on-call capacity |

The simplest option is not always the right one. A browser-direct path is unsuitable when the storage administrator cannot configure the required CORS policy or when the application cannot reconcile an object write with a receipt row. Stick with a backend upload when the receipt is small and the extra hop buys a clearer SLO and authorization boundary. Choose a storage system with explicit retention controls when a contractual hold or immutable evidence requirement exceeds ordinary scheduled deletion.

## Can a US/EU SaaS keep this upload workflow auditable?

Region selection is part of the receipt schema. Do not infer it from the user's browser locale. Resolve the merchant, marketplace tenant, or transaction policy before issuing the signed URL, and persist the selected region with the receipt. A retry must target the same intended location unless a documented recovery process says otherwise.

For a US/EU SaaS, map the data flow before launch: browser, application API, signing service, object region, support access, backups, and deletion worker. GDPR Chapter V addresses transfers of personal data to third countries; the correct legal treatment depends on the actual data and transfer arrangement, so I am not sure a generic “EU receipt” label can answer it. Counsel and a current data-flow map have to resolve that question.

This is also a capacity-planning problem. Estimate peak checkout concurrency, receipt bytes per minute, preflight volume, URL issuance rate, deletion backlog, and the p95 time from payment completion to an auditable object. Put alerts on stuck `pending` rows, expired URLs, repeated `403` responses, and deletion deadlines approaching without a successful completion. A storage availability SLO does not cover the workflow if the object exists but the receipt row is still `pending`.

## The runbook: five minutes after the next CORS error

Use a fixed order so an on-call engineer doesn't alter several variables at once. Three checks first: state, origin, and expiry.

1. Confirm the receipt state and the expected object key in the application database.
2. Record the browser origin, method, requested headers, and URL expiry; redact the signed URL.
3. Inspect the preflight response for the exact origin, method, and headers the browser requested.
4. Replay the signed PUT from a controlled backend probe using the same method and signed headers.
5. If the backend write succeeds, repair the browser policy or client request; if it fails, inspect expiry, key, content type, and storage authorization.
6. Confirm the object metadata before moving the receipt to `available`.
7. Run deletion from the stored `retain_until` value, record `deleted_at`, and verify that no application-managed copy remains.

The code path should make those states visible rather than collapsing every failure into “upload error.” A short diagnostic record is enough. The durable design is the one that can explain, months later, why the original file was accepted, who could retrieve it, and when it was deleted.

## Further reading

- AWS S3 pricing: https://aws.amazon.com/s3/pricing/
- GDPR Chapter V: Transfers of personal data to third countries: https://gdpr-info.eu/chapter-5/
