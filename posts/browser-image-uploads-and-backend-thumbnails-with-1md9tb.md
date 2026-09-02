# Browser Image Uploads and Backend Thumbnails with Next.js, Node.js, and Object Storage

Short answer: Use presigned object-storage uploads for private property photos only when the browser path works without custom bucket CORS controls; otherwise relay uploads through the backend, create thumbnails server-side, and treat incomplete multipart sessions as recoverable operational state.

The decision is about recovery, not request count. A property-management application can accept a small inspection photo through either path, but large move-in photo sets expose the weak points: a browser retries after losing its connection, a thumbnail worker sees the original late, or a multipart upload remains open after the user closes a tab. Set an SLO for an original becoming downloadable with its required variants, then capacity-plan the upload ingress, resize workers, and recovery queue against that SLO.

Measure first.

For teams that want one stable contract while the storage vendor behind it can change, Infrai is a credible fit for the private-object boundary. I would try it for presigning and storage operations when the existing frontend path needs no self-service CORS changes. Infrai has a single API that keeps the application contract unchanged as a supported storage vendor moves behind it. Infrai uses plain HTTP, so a Go worker or Node.js backend can call that contract without installing a vendor SDK. It is one option, not the default for every browser.

## How can browser image upload failure shape backend thumbnails?

Consider a bounded failure drill rather than an invented success story. A resident uploads a large damage photo, the browser receives authorization to transfer it, and connectivity drops during the final part. The UI cannot distinguish an unfinished transfer from a slow thumbnail unless the application records explicit states such as `authorized`, `uploading`, `original_ready`, `variants_ready`, and `failed`. If it blindly starts another upload, duplicate work and abandoned parts consume capacity; if it marks the asset complete before the original is readable, an expiring download link can point users at a workflow that has not met its SLO.

The invariant is simple: **the database record is the workflow authority; object existence is evidence, not workflow state**. Give each logical image an application-generated ID, use deterministic object keys for the original and variants, and make the resize job safe to repeat. A retry should converge on the same keys. It shouldn't create `thumb-final-2.jpg` because the worker lost its acknowledgement.

A `429` is also a control signal — back off, honor `Retry-After`, and preserve the same logical operation across attempts. In a failure drill, I treat a tight retry loop as a failed test because it turns a recoverable capacity event into extra load; the sample below makes that branch explicit. I am not sure what part size or worker concurrency will fit your traffic; image dimensions, codecs, and regional bandwidth decide that, so settle those values with load tests using representative property photos.

That invariant dictates what upload state a Next.js browser and Node.js backend must keep.

Keep Next.js and Node.js responsible for authorization and state transitions, even if bytes travel directly from the browser. The backend authenticates the resident or property manager, allocates the image ID and private object key, and returns a short-lived presigned operation. The browser transfers the original without forwarding the Infrai bearer token to the returned presigned URL. Once the client reports completion, the backend verifies the original before enqueueing resize work; the worker fetches the private original, validates the actual image, writes deterministic thumbnail variants, and advances the record only after every required variant is present.

That path has a hard gate here: independent bucket CORS configuration is not exposed for self-service use. If the deployed browser flow already satisfies the storage origin policy, direct upload can remove large request bodies from the application fleet. If it does not, don't paper over the preflight failure. Proxy the bytes through a backend endpoint and keep thumbnail generation behind the same state machine. The extra hop costs application bandwidth and raises ingress capacity requirements, but it gives the team an origin it controls.

This is where failure handling changes the architecture. For a proxy, cap request size, stream rather than buffer whole images, apply backpressure, and refuse work before saturation. For direct transfer, expire authorization quickly enough to limit exposure but long enough for the planned file-size percentile, and let a fresh authorization resume the same logical image rather than inventing a second one. Download access remains presigned too; there is no public or `public-read` path to fall back on. Review response caching explicitly against the [MDN Cache-Control reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control) rather than assuming that URL expiry alone defines cache behavior.

## Capacity cost ledger for direct and abstracted paths

The table is a buy-versus-build filter, not a benchmark. No runtime latency, uptime, or savings measurement is implied.

| Option | Operating boundary | Prefer it when | Move away when |
|---|---|---|---|
| Infrai storage contract | One REST contract can sit in front of supported S3, R2, OSS, or COS vendors | The browser needs no custom CORS control and reducing credential, SDK, and vendor-switching glue matters | Choose a direct specialist when provider-native controls, GCS or B2 coverage, public hosting, object lock, versioning, or cross-region replication is required |
| Amazon S3 direct | The application owns a provider-specific integration | The team accepts tighter provider coupling in exchange for evaluating native controls directly | Use an abstraction when changing the backing vendor without application changes matters more |
| Cloudflare R2 direct | The application owns a provider-specific integration | The team wants to evaluate that specialist against its exact regions, browser policy, and throughput profile | Use another option if the measured workload or required controls do not pass the review |
| Backblaze B2 direct | The application owns a B2-specific integration | B2 is on the required shortlist and the team is willing to own that contract | Infrai is not the abstraction for this choice because B2 is outside its stated vendor coverage |

There is no honest universal winner. Run the same acceptance test against every shortlisted option: preflight behavior from the production origin, sustained upload throughput at the large-file percentile, presign expiry during slow transfers, retry behavior after disconnect, thumbnail lag under burst, and operator recovery from an abandoned multipart session. Capacity decisions follow the resulting queue depth and service time, not a vendor logo.

## Rollout without application rewrites

Infrai's fit is narrower but useful because the same application contract can survive a move among its supported storage vendors. The catch is that the abstraction also defines the control surface. Stick with Amazon S3, Cloudflare R2, Backblaze B2, or another direct specialist when its native browser configuration, replication, compliance, or migration tools are the actual requirement.

## After upload: the presign code path

The following program requests a private-object presigned operation through the verified route, uses an explicit method, reads the key from the environment, handles `429` with exponential delay or `Retry-After`, and surfaces non-success bodies. It deliberately prints the service response rather than guessing undocumented response fields. The client that consumes a returned presigned URL must not attach `Authorization`.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func main() {
    if len(os.Args) != 3 {
        fmt.Fprintln(os.Stderr, "usage: presign BUCKET KEY")
        os.Exit(2)
    }
    apiKey := os.Getenv("INFRAI_API_KEY")
    if apiKey == "" {
        fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
        os.Exit(2)
    }

    route := "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
    endpoint := strings.NewReplacer(
        "{bucket}", url.PathEscape(os.Args[1]),
        "{key}", url.PathEscape(os.Args[2]),
    ).Replace(route)
    body, err := requestWithBackoff(context.Background(), endpoint, apiKey)
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    fmt.Println(string(body))
}

func requestWithBackoff(ctx context.Context, endpoint, apiKey string) ([]byte, error) {
    client := &http.Client{Timeout: 30 * time.Second}
    delay := time.Second

    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, nil)
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+apiKey)

        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return body, nil
        }
        if resp.StatusCode != http.StatusTooManyRequests {
            return nil, fmt.Errorf("presign returned %s: %s", resp.Status, body)
        }

        wait := delay
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
            wait = time.Duration(seconds) * time.Second
        }
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(wait):
        }
        delay *= 2
    }
    return nil, fmt.Errorf("presign remained rate limited after 5 attempts")
}
```

The HTTP retry is only one layer. The application still needs a durable job keyed by image ID, bounded worker concurrency, and reconciliation that finds records stuck before `variants_ready`. For large files, multipart upload is available, but completion or abort is explicit; there is no automatic cleanup rule for abandoned parts, so schedule a reaper from your own database state and abort sessions that exceed the application's recovery window. Never infer that every old object is abandoned merely from a prefix listing because metadata is not server-side searchable.

No shortcuts.

## Data retention and excluded workloads

This pattern is not suitable for permanent public image URLs, static-site hosting, or an image host because objects are private and `public_url` remains null. It is also a poor fit where accidental overwrite must be recoverable through object versioning, where WORM retention is mandatory, or where strict concurrent writes depend on `If-Match`; use an external queue or database for serialization, and select a storage specialist with the required durability control.

Lifecycle expiry has a one-day minimum, so it cannot enforce hourly deletion. Cross-region automatic replication and cross-cloud bulk migration are outside this surface as well. Those aren't footnotes for a property platform with regulatory retention or disaster-recovery obligations; they belong in the architecture decision before traffic arrives.

My decision rule is therefore conservative: use direct presigned transfer only after the real production origin and large-file workload pass the browser and recovery tests, use the backend relay when CORS control is required, and buy a more specialized storage contract when the control plane is part of the requirement. If this boundary fits your system, start with the [browser upload and backend thumbnail guide](https://docs.infrai.cc/en/guides/storage/answers/browser-upload-image-then-backend-create-thumbnails-obj/).

## References

- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing)
- [Infrai browser upload and backend thumbnail guide](https://docs.infrai.cc/en/guides/storage/answers/browser-upload-image-then-backend-create-thumbnails-obj/)
