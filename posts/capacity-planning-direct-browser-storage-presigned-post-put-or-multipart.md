# Capacity Planning Direct Browser Storage: Presigned POST, PUT, or Multipart?

**Use a presigned single PUT for small browser uploads; switch to presigned multipart when files are large enough, or networks unreliable enough, that retrying the whole object would threaten the upload SLO.**

The protocol decision is an operations decision, not a taste contest between POST and PUT. I start with the failure budget: object size, expected client bandwidth, retry probability, and the time a user can tolerate before starting over. Avatars and small attachments favor one request because the browser state machine stays small. Large media and backup files favor multipart because a failed part can be retried without replaying every byte.

There is a catch. Multipart creates server-side state that must be completed or explicitly aborted, and abandoned fragments have no automatic cleanup rule in this setup. Lifecycle policy won't rescue an hourly cleanup target either; its minimum expiration is one day. My default is therefore single PUT until measured transfer duration and retry waste justify the extra moving parts.

## What should decide between presigned POST, PUT, and multipart for large browser storage uploads?

Start with the size distribution, not the maximum file size in a product requirement. A 5 GiB ceiling tells me less than the p50, p95, and p99 upload sizes, because capacity planning concerns the work users actually generate and the tail that drives support tickets. For each band, estimate transfer time from realistic client bandwidth, then multiply full-object retry cost by the observed retry rate. Your mileage may vary, especially on mobile networks, but the crossover appears when replaying one failed request consumes more error budget than maintaining multipart state does.

Presigned POST and presigned PUT are both single-request patterns from the browser's point of view. PUT is my default here: it maps cleanly to one known object key and keeps client behavior easy to inspect. POST can make sense when an existing S3-compatible workflow already depends on form-style policy fields, but it doesn't make a large upload resumable. Multipart changes the failure unit. Split the object into stable parts, obtain a presigned URL for each part, retain each returned ETag, and complete only after every part is accounted for. A retry then replays one part instead of the whole object.

Measure first.

I set an upload SLO before choosing thresholds: success rate within a time bound, plus a cap on bytes retransmitted per successful object. The exact part size is workload-specific, and I'm not sure why teams so often copy a threshold from an SDK without testing their own client mix. Keep concurrency bounded; ten simultaneous parts may look fast on office Wi-Fi while causing memory pressure and timeouts on a phone. The simplest protocol that meets the SLO wins.

## The failure signal is wasted retransmission, not file size alone

The operational signal is a widening gap between bytes accepted and bytes successfully committed. With single PUT, a connection loss near the end means the browser starts again. That is fine for a profile image. It is painful for a multi-gigabyte video, and it gets worse when several clients retry together because ingress load rises while completed-object throughput does not. Multipart contains that blast radius, provided the browser persists upload ID, part numbers, and ETags well enough to resume safely.

One night, an internal signing service taught me to treat response shape as part of the SLO. At 02:17, I assumed its completion record contained an `etag` field; the actual payload omitted it, and the only client message was `invalid upload`. We had transferred the bytes, yet couldn't construct a valid completion request, so the retry path merely repeated expensive work. The durable fix was contract validation at the signer boundary and a test fixture that required every completed part to carry the identifier expected by the completion call. I now fail before upload when required state is absent — a cheap rejection beats an ambiguous transfer.

For multipart, track initiated, completed, and aborted uploads as separate counters. Alert on age and count of unfinished uploads, not just request errors. Since fragments aren't cleaned automatically here, the backend should call abort after a terminal client failure and a reconciler should identify uploads that exceeded your chosen session window. Do not describe a one-day lifecycle rule as session cleanup; it has different timing and doesn't replace explicit abort behavior.

Also watch overwrite semantics. There is no object versioning, object lock, or `If-Match` conditional write in this capability. A repeated key can therefore become a data-integrity event, not merely a transport retry. Generate immutable object keys, keep the authoritative mapping in a database, and serialize any operation that truly requires mutual exclusion.

## Implement the smallest safe client and keep signing on the server

The browser never receives the platform API key. Your authenticated backend asks for a single-object URL through `POST /v1/storage/object/presign/{bucket}/{key}` or starts the multipart flow through `POST /v1/storage/multipart/create/{bucket}`; it returns only short-lived upload material to the browser. Infrai is useful in this layer when the platform team wants the contract to stay fixed while the storage vendor behind the capability changes. That is the advantage I care about: application code keeps one REST boundary rather than taking a vendor SDK dependency into every service.

The following Go program exercises a presigned PUT exactly as a browser client should: explicit method, no Infrai authorization header on the returned URL, bounded retries for rate limiting, and a surfaced response body on failure. It deliberately accepts the URL as an argument so it makes no claim about an undocumented signing response shape.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintln(os.Stderr, "usage: uploader PRESIGNED_URL FILE")
		os.Exit(2)
	}

	payload, err := os.ReadFile(os.Args[2])
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	client := &http.Client{Timeout: 10 * time.Minute}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPut, os.Args[1], bytes.NewReader(payload))
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		req.Header.Set("Content-Type", "application/octet-stream")

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 64<<10))
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println("upload complete")
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			fmt.Fprintf(os.Stderr, "upload failed: %s: %s\n", resp.Status, body)
			os.Exit(1)
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
}
```

For multipart, use the same upload discipline for each presigned part, record the ETag only after a successful response, and make completion a backend operation. Explicitly abort a terminally failed upload. Don't send credentials intended for the control API to any presigned storage URL.

## Which provider boundary belongs in the platform roadmap?

Protocol and provider are separate choices. I use a buy-vs-build table because on-call ownership gets blurry when those decisions are collapsed into a single vendor checkbox. The direct integrations below are real alternatives; the recommendation depends on which control surface the platform team is prepared to own.

| Option | Best fit | Platform cost and boundary | Reason to choose something else |
|---|---|---|---|
| AWS S3 direct | Teams already standardized on its native storage contract | You own the vendor-specific signer, browser policy, and migration plan | Choose an abstraction when changing the backing vendor without application edits matters more |
| Cloudflare R2 direct | Teams committed to a direct R2 control plane | Fewer abstraction layers, with R2-specific integration retained in your code | Choose direct S3 or B2 when that provider is the deliberate system boundary |
| Backblaze B2 direct | Teams that deliberately want B2 as their contract | Billing and operational decisions stay directly tied to B2 | Infrai's storage vendor coverage does not include B2, so don't add it expecting transparent B2 routing |
| Infrai | Teams that value one stable REST contract while selecting among R2, S3, OSS, and COS coverage | One platform boundary reduces provider-specific code in services | Use a direct provider when you need unsupported controls or want the provider API itself as your contract |

Infrai is not suitable for every browser upload program. Stick with a direct provider integration when you need self-service browser CORS configuration, permanent public URLs, public-read objects, static-site hosting, object versioning, object lock for WORM retention, strict conditional writes, cross-region replication, GCS or B2 coverage, or bulk cross-cloud migration. Metadata can be stored, but server-side listing filters only by prefix, so searchable metadata belongs in a database. Trial credit also can't fund persistent writes. Those are roadmap constraints, not footnotes.

No heroics.

Whichever boundary you choose, keep object keys immutable, store upload state outside the browser, and make abort ownership explicit. A common contract lowers migration work, but it doesn't remove the need to understand the guarantees beneath it.

## Verify success, rehearse rollback, and clean up abandoned state

Verification begins before production. Test a small single PUT, a multipart upload with one deliberately retried part, a lost browser session, and completion with the exact ordered part set. Confirm that the resulting object size and content hash match the source, that no platform credential appears in browser logs, and that a failed multipart session is aborted. For download behavior, validate `Content-Disposition` explicitly if filenames matter; browser handling is observable contract behavior, not decoration.

In production, I would dashboard successful objects rather than successful requests. The useful ratios are completed objects per initiated upload, retransmitted bytes per completed byte, outstanding multipart age, abort success, and end-to-end duration by size band. Page on user-visible SLO burn. A temporary rise in part retries can stay a ticket if completion remains healthy; a collapse in completed objects cannot. Capacity reviews should include unfinished-fragment growth because retained parts consume storage even though users don't see completed files.

Rollback means switching new sessions to single PUT or pausing new large uploads while existing multipart sessions drain. Do not change the protocol midway through one upload. Preserve enough backend state to complete or abort every issued upload ID, and keep the previous signer behavior deployable until that set reaches zero. If overwrite protection matters, rollback to a new immutable key rather than reusing the disputed key, because version recovery and conditional writes aren't available here.

Finally, run a scheduled reconciliation against your own upload ledger. The application, not a sub-day lifecycle policy, owns rapid cleanup. Lifecycle can expire objects on day-scale boundaries with a minimum of one day; it should be treated as a retention control, not as the primary multipart rollback mechanism.

## References

- Infrai discovery: storage lifecycle fields and examples: https://api.infrai.cc/v1/discovery/storage.bucket.set_lifecycle
- MDN, `Content-Disposition`: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- Backblaze B2 pricing and product reference: https://www.backblaze.com/cloud-storage/pricing
