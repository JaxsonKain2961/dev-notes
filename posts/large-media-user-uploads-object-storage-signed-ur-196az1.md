# Large-Media User Uploads: Object Storage, Signed URLs, and Server Proxy Capacity

Short answer: for a beginner SaaS handling user avatar uploads, let the browser send bytes to private object storage through a short-lived signed URL, and keep the application server responsible for identity, object naming, and the final database decision. Use a server proxy when the product must inspect or transform every byte before storage. For a media product, this boundary matters because large-file throughput is a capacity problem long before it becomes a framework problem.

The useful distinction is not “modern” versus “simple.” It is which tier owns the upload body. A proxy makes the Node.js service the byte-moving component: it consumes connection slots, bandwidth, memory or streaming buffers, request time, and the same operational budget used by ordinary SaaS requests. A direct transfer moves that pressure to the storage path, while the app still controls who may upload what and where.

That is the runbook decision. The URL is a temporary capability, not a user record and not an authorization shortcut.

## How should a beginner SaaS handle avatar upload, object storage, signed URLs, and a server proxy?

Start with a private bucket and an application-owned object key. Authenticate the user in the app, derive a key such as `users/{id}/avatar.jpg`, and issue a signed upload operation for that exact destination. The browser uploads the selected file directly. After the transfer, the app records the key, or a pending-upload record, according to its completion rule. When the avatar is displayed, the app creates a fresh signed read rather than storing an expiring URL in the user row.

That flow is not permissionless just because the browser sees a URL. The server should never accept an arbitrary account ID or arbitrary storage path as the authority for the write. It should derive both from the authenticated session, constrain the expected size and media type at the application boundary, and treat a client-reported “done” event as a signal to verify rather than as proof of ownership.

A proxy is the better fit when synchronous inspection or byte transformation is a product requirement. A moderation step that must happen before persistence, or a required image rewrite that cannot be deferred, gives the proxy a real job. The catch is that the job must be capacity-planned. A 413 response means the request policy rejected the body; it is not evidence that storage is healthy, and it should be visible separately from a timeout, a failed signed transfer, or an authorization denial.

For a profile image, last-write-wins may be acceptable. For media ingest with review, it usually is not. Make that a database decision, not an accidental property of whichever upload finishes last.

## What fails first when large-file throughput meets a Node.js upload path?

The failure mode I look for first is queueing at the application tier. A server proxy can stream instead of buffering the whole file, but streaming does not erase open connections, socket pressure, TLS work, request duration, egress, or the need to protect regular API traffic from a burst of large bodies. If the same process serves login, billing, and avatar upload, the upload route is now part of every other route’s SLO story.

Direct transfer has a different failure shape. The application can issue more upload tickets than it receives completed files because users close a tab, lose connectivity, or choose another image. A successful object write can also arrive after the user has started a newer upload. Therefore, measure at least three transitions: signed operations issued, storage transfers observed, and object keys committed as current. Alert on a sustained gap between them, with labels for tenant, size band, and client version. A single “upload success” counter hides too much.

The throughput budget should be stated before implementation. Estimate concurrent uploads, the largest permitted body, expected completion time, and the fraction of application capacity reserved for ordinary requests. Then test with realistic network constraints and a mix of small avatars and large media files. A benchmark that uploads one file at a time proves almost nothing about the service’s worst queue. For a concrete review, I write down the path of one upload from the authenticated request to the final profile update, then mark every place where bytes, metadata, or authority changes hands: the browser selects a file, the app issues a constrained capability, storage receives the body, the app confirms the resulting object, and the database decides whether that object is current. That map exposes a common capacity mistake: engineers count only the storage transfer and forget the proxy’s connection lifetime, the confirmation traffic, abandoned candidates, retry storms, and cleanup work. It also makes load testing more honest, because a test can vary file size, cancellation point, and concurrent users while checking that the ordinary API SLO stays inside its budget.

I keep rollback explicit. If the canonical key is overwritten, the old bytes may be gone; if each candidate gets a new key, the database can move the current pointer back while an asynchronous cleanup policy removes abandoned candidates. Object lifecycle rules are useful for retention, but a lifecycle policy should not be treated as a same-minute garbage collector. The operational owner still needs a cleanup path for pending objects and a report of what it removed.

There is one uncomfortable uncertainty in any client-direct design: the browser and the storage service can disagree about what the user believes happened. I'm not sure a client-only callback can resolve that disagreement for a workflow that matters, so I use a server-side confirmation state and make the visible avatar change depend on it. Your mileage may vary if the product is disposable and the image is not security-sensitive, but the state model should be a deliberate choice.

## The safe implementation boundary is small, but it is not casual

The handler that creates a signed operation should be boring. It authenticates, validates policy, chooses the key, and returns a temporary capability. It does not proxy media bytes, trust a browser-selected path, or persist a temporary URL as if it were an object identity. The following Go helper isolates the part that must not drift into request data; the surrounding service can call its storage provider using that provider’s documented signing mechanism.

```go
package avatar

import (
	"fmt"
	"strings"
)

type UploadPlan struct {
	Key       string
	MaxBytes  int64
	MediaType string
}

func PlanAvatarUpload(authenticatedUserID string, maxBytes int64) (UploadPlan, error) {
	if strings.TrimSpace(authenticatedUserID) == "" {
		return UploadPlan{}, fmt.Errorf("authenticated user ID is required")
	}
	if maxBytes <= 0 {
		return UploadPlan{}, fmt.Errorf("max upload size must be positive")
	}

	return UploadPlan{
		Key:       "users/" + authenticatedUserID + "/avatar.jpg",
		MaxBytes:  maxBytes,
		MediaType: "image/*",
	}, nil
}
```

The important behavior is the input boundary, not the language. The object key comes from authenticated identity, and the policy is represented before a signing request is made. In a real handler, persist a pending record with an expiry and an upload token or version, then accept the object as current only when the confirmation checks match that record. If two browser tabs race, compare versions or use a transaction; do not infer correctness from request arrival order.

Keep the storage path private unless the product explicitly needs public delivery. For a private avatar, a signed read should be generated at access time and kept out of durable profile data. If the browser downloads rather than displays the response, verify the intended `Content-Disposition` behavior: inline display and attachment download are different user experiences, and the HTTP response header is the place to express that intent.

Three words: protect the API.

Use separate concurrency and bandwidth limits for the proxy route if a proxy is unavoidable. Put upload duration, body size, status class, and cancellation reason in metrics. A timeout before the first byte, a timeout during transfer, and a rejected media type need different remediation. The on-call page should tell the engineer which boundary failed, not merely that “avatars are broken.”

## Which path should the team deploy, verify, and roll back?

Use the following decision table during design review. It keeps the comparison on operating responsibility rather than on a fashionable API shape.

| Path | Suitable when | Main SLO and capacity cost | Rollback posture |
|---|---|---|---|
| Signed browser-to-storage transfer | The app can authorize intent before bytes arrive and can confirm completion afterward | Storage and client transfer become the main throughput variables; the app must reconcile abandoned tickets | New candidate keys make pointer rollback straightforward; cleanup must be planned |
| Application server proxy | Every byte needs synchronous inspection or transformation | The app owns upload bandwidth, open connections, request duration, and isolation from normal API traffic | The app can reject before persistence, but retries and partial streams need careful handling |
| Hybrid quarantine flow | Inspection is required, but it can happen after the initial transfer | The app tracks a pending state and an inspection queue; user-visible completion waits for promotion | Keep the candidate separate until the database promotes it; expire rejected candidates |

Verification should exercise the uncomfortable paths: an expired capability, a user uploading to a key they do not own, a body over the policy limit, a canceled connection, two concurrent uploads, and a storage write that completes after the browser has navigated away. Verify that no profile row stores a temporary URL, that an uncommitted object is not shown as current, and that a rejected candidate reaches a known retention path.

For deployment, canary the direct-transfer route with a size distribution that resembles production rather than only tiny test images. Watch the ratio of issued tickets to committed keys, p95 and p99 transfer duration, application request saturation, and cleanup backlog. Define rollback as a feature-flag change plus a policy for objects already uploaded through the new path. Switching the UI back to a proxy without handling those objects can create duplicate media or leave the old path unable to find the new key.

The direct path is not suitable when the application must make a synchronous byte-level decision before storage, when the storage service cannot meet the required private-delivery behavior, or when the team cannot operate reconciliation and cleanup. Stick with a proxy or a quarantine design in those cases. Conversely, a proxy is not suitable for a large-file workload whose main requirement is to keep application capacity available and whose validation can happen before authorization or after a controlled transfer.

There is no universal winner. The durable choice is the one whose failure modes have an owner, an SLO, and a rollback procedure.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
