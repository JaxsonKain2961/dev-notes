# Browser 404 Triage After Export Jobs: Audit the Download Link and Object Key

Short answer: don't mark an export complete until the exact object key saved with the job can be read and used to create the browser download link. Treat object creation and link publication as separate commit points; a 404 then becomes a bounded state-machine failure, not a vague storage-consistency theory.

A useful first move is to stop retrying the browser and inspect three values from one job record: the key written by the exporter, the key passed to the link signer, and the key requested by the browser. Those byte strings must match. If they do, test whether the object is readable before investigating URL expiry, proxy rewriting, or authorization. This ordering matters because repeated browser retries can hide timing bugs without identifying which boundary violated the delivery SLO.

## How should you troubleshoot a browser download 404 after an export job?

Start at the state transition, not at the browser. The job should move through something like `queued -> rendering -> uploading -> verifying -> ready`; only `ready` may expose a link. If `complete` currently means that rendering ended, the label promises too much. The upload may still be open, a multipart completion step may still be pending, or the database may contain a key assembled by a second code path.

For one failing job, capture a compact evidence bundle: job ID, state-transition timestamps, bucket or namespace, persisted object key, object size, checksum or generation identifier when available, signer input, link issuance time, and the final HTTP status. Don't log the signed query string because it is a credential. Record a hash of the full URL if correlation requires one.

Then classify the failure in this order:

1. **Wrong key or prefix.** Compare escaped strings, including leading slashes, repeated separators, case, spaces, and tenant prefixes. An object named `exports/acme/42.csv` is different from `/exports/acme/42.csv`. Listing by a familiar prefix is weaker evidence than looking up the persisted key directly.
2. **Publication race.** Check whether `ready_at` precedes the successful upload completion and direct read probe. A queue acknowledgement or a closed local file doesn't prove that a remote object is readable.
3. **Link construction.** Generate a fresh link from the persisted key and compare its path with the browser request path. Reverse proxies and application routers can decode or normalize a path differently, so keep the object key as data rather than splicing it into several URL templates.
4. **Expiry or access policy.** If direct lookup succeeds but the browser request fails, inspect signing time, expiry, clock skew, method, and response headers. A public-facing 404 may intentionally reveal less than the internal authorization result.

Don't guess.

The phrase "eventual consistency" is often used too early. Consistency behavior depends on the storage system and operation, and I'm not sure which guarantee applies unless the selected service's current documentation names the exact write, overwrite, list, and read operations. Resolve that uncertainty from the service contract; meanwhile, a verification state makes the application correct under a weaker visibility assumption and produces evidence when the assumption is wrong.

## Make the object key a durable contract

A key should be constructed once, before upload, then persisted with the job and passed unchanged to upload, verification, and signing. Rebuilding it later from a display filename, current date, tenant slug, or mutable export settings creates multiple authorities. The resulting 404 looks intermittent because only certain names or retries cross the divergent branch.

Use an opaque, immutable key for delivery and retain the friendly filename as response metadata. A shape such as `exports/<tenant-id>/<job-id>/artifact` is easier to reason about than one derived from user text. If an extension is operationally useful, normalize it in the same function and test the returned string as a value. The important property isn't this particular prefix; it is that every stage consumes the stored result.

The following Go sketch keeps storage details behind a small interface. `CompleteUpload` represents the storage system's required finalization operation; for a multipart workflow, successful part uploads alone are not publication. The transition to `ready` occurs only after a lookup of the exact key returns the expected identity and size.

```go
package exports

import (
    "context"
    "errors"
    "fmt"
    "path"
    "time"
)

type ObjectInfo struct {
    Size       int64
    Generation string
}

type Store interface {
    CompleteUpload(ctx context.Context, uploadID, key string) (ObjectInfo, error)
    Stat(ctx context.Context, key string) (ObjectInfo, error)
    SignRead(ctx context.Context, key string, expiresAt time.Time) (string, error)
}

type Job struct {
    ID               string
    TenantID         string
    ObjectKey        string
    ExpectedSize     int64
    ObjectGeneration string
    State            string
    ReadyAt          time.Time
}

func NewObjectKey(tenantID, jobID string) (string, error) {
    if tenantID == "" || jobID == "" {
        return "", errors.New("tenant and job IDs are required")
    }
    key := path.Join("exports", tenantID, jobID, "artifact")
    if key == "." || key[0] == '/' {
        return "", errors.New("invalid object key")
    }
    return key, nil
}

func Publish(ctx context.Context, store Store, job *Job, uploadID string, now time.Time) (string, error) {
    completed, err := store.CompleteUpload(ctx, uploadID, job.ObjectKey)
    if err != nil {
        return "", fmt.Errorf("complete upload: %w", err)
    }

    visible, err := store.Stat(ctx, job.ObjectKey)
    if err != nil {
        return "", fmt.Errorf("verify object: %w", err)
    }
    if visible.Size != job.ExpectedSize || visible.Generation != completed.Generation {
        return "", errors.New("object identity verification failed")
    }

    job.ObjectGeneration = visible.Generation
    job.State = "ready"
    job.ReadyAt = now

    link, err := store.SignRead(ctx, job.ObjectKey, now.Add(15*time.Minute))
    if err != nil {
        return "", fmt.Errorf("sign read link: %w", err)
    }
    return link, nil
}
```

In production, the job update and link response also need a clear durability boundary. Persist `ObjectKey` before work starts, and persist the verified generation plus `ready` state before returning a URL. If the database write fails after the object is complete, retry the state transition idempotently against the same key; don't create a second key. The retry should regard an already completed object with the expected generation and size as success.

A multipart upload deserves special care because parts are intermediate resources, not the final downloadable object. Track the upload identifier, complete or abort it deliberately, and add cleanup for abandoned uploads. The SRE concern is capacity as much as correctness: unfinished parts consume storage while offering no valid download, so measure their count and age rather than waiting for a cost surprise.

## Set an SLO around the handoff, not the worker

A job-runtime metric can be green while users receive 404 responses. The user-visible indicator is closer to "exports that become downloadable within the promised interval and remain downloadable for the advertised link lifetime." Split it into rendering, upload, verification, database publication, and first-read latency so an alert points to a boundary.

Capacity planning follows from those stages. Estimate peak exports per minute, output-size distribution, concurrent upload memory, multipart part count, verification request rate, retry amplification, and link-generation load. A verification call per export is usually predictable; an unbounded client retry loop is not. Put a capped retry policy with jitter around transient verification failures, leave the job in `verifying`, and publish no link until success. A deadline should move the job to a retryable terminal state with an internal reason, never to `ready`.

A buy-versus-build decision should include on-call load and exit cost, not merely request price:

| Decision area | Managed object storage | Self-hosted object storage |
|---|---|---|
| Consistency contract | Read the documented guarantees for each operation | Define, test, and operate the guarantee across replicas |
| Multipart lifecycle | Configure lifecycle and monitor abandoned uploads | Build cleanup, capacity alarms, and recovery procedures |
| Signing and key management | Integrate the service's signing and identity controls | Operate key rotation, signing, and audit paths |
| Failure ownership | Provider handles the storage control plane; the application still owns publication ordering | The platform team owns both control plane and application ordering |
| Portability | Adapters and stored object identities reduce coupling | Protocol compatibility helps, but operational semantics still need tests |

The catch is that the verification gate adds a storage lookup and some latency. It is not suitable when a response must stream immediately without durable storage; use a streaming download path with its own backpressure and retry semantics in that case. For durable asynchronous exports, skipping verification trades a small, measurable handoff cost for an ambiguous failure that reaches the user and wakes the on-call engineer. That's a poor bargain.

## Verify the fix and keep rollback boring

Test the state machine with a controllable fake store before changing production traffic. Hold `Stat` unavailable after upload completion and assert that no link is returned and the job remains `verifying`; return a mismatched size or generation and assert the same. Feed keys containing spaces, mixed case, Unicode, and leading-slash attempts through the one key constructor. Also test retry after a database failure: the same job must converge on the same object identity without duplicate publication.

Then canary the change. Measure `upload_complete_to_ready` latency, verification attempts, jobs stuck in `verifying`, first browser read outcomes, and abandoned multipart uploads. Sample successful cases as well as errors so absence of 404 logs doesn't masquerade as health. The acceptance check is simple: a `ready` record identifies one readable object generation, and a newly issued link retrieves that generation.

Rollback should disable link publication while leaving completed objects intact. Keep the previous worker available, but don't restore the old behavior of declaring readiness before verification; instead, pause new exports or leave them queued while the canary is reverted. Because the object key and generation are durable, operators can replay verification and safely publish links after recovery. Document how to abort old multipart uploads only after confirming that no active job references their upload IDs.

One sharp alarm beats six vague ones: page on sustained SLO burn for user-visible delivery, and send growing verification latency or abandoned-upload age to a lower-urgency operational queue until it threatens that SLO. Your mileage may vary with export size and storage semantics, so set thresholds from observed distributions rather than copying a universal timeout.

## Further reading

The primary material for multipart finalization and storage-operation contracts is listed below.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://cloud.google.com/storage/docs
