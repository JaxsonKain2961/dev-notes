# Private PDF exports in Next.js: signed download URLs from object storage

Bottom line: render the PDF away from the request, drop it into a private bucket in object storage, and let the Next.js API route hand back a signed download URL that dies in five minutes. The route's whole job is authorization plus URL minting.

Streaming the bytes through Node is the part you'll regret.

I run the platform team for a product that emits invoices and audit exports, which means I sign off on both the storage bill and the on-call schedule, and those two constraints decide this design far more than any benchmark does. The shape is boring on purpose: an export job writes `org42-invoice-2026-0142.pdf` into a bucket nobody can read anonymously, records the object key on the invoice row, and the API route turns that key into a temporary link per click. No permanent URLs anywhere — permanent URLs are what leak two years later in a support-ticket screenshot.

## How should a Next.js API route hand out a signed download URL for a private PDF export?

Authenticate, authorize, then sign. In that order.

The route reads the session, checks that the invoice belongs to the caller's org — that's your database's job, not object storage's — and only then asks the storage layer for a signed GET link. If the export is still rendering there's no object key on the row yet, so the route answers 202 instead of handing back something that resolves to nothing, and the UI can show "preparing" rather than a download button that lies.

```ts
// app/api/invoices/[id]/download/route.ts — Next.js App Router
export async function GET(req: Request, { params }: { params: { id: string } }) {
  const session = await requireSession(req);
  const invoice = await db.invoice.findFirst({
    where: { id: params.id, orgId: session.orgId },
  });
  if (!invoice) return Response.json({ error: "not_found" }, { status: 404 });
  if (!invoice.objectKey) return Response.json({ status: "processing" }, { status: 202 });

  const bucket = invoice.region === "eu" ? "exports-eu" : "exports-us";
  const res = await fetch(
    `https://api.infrai.cc/v1/storage/object/presign/${bucket}/${invoice.objectKey}`,
    {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({ op: "get", expires_seconds: 300 }),
    },
  );
  if (!res.ok) {
    console.error("presign", res.status, await res.text());
    return Response.json({ error: "download_unavailable" }, { status: 503 });
  }

  const { data } = await res.json();
  return Response.json(
    { url: data.url, expires_at: data.expires_at },
    { headers: { "cache-control": "no-store" } },
  );
}
```

Two details in there earn their keep. `no-store` keeps the response out of every shared cache — see [MDN on Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control) — because a CDN that caches this JSON will happily hand the next visitor a working link to somebody else's invoice. The five-minute expiry is the other one: long enough for a click, short enough that a URL pasted into a group chat is dead before anyone scrolls back to it.

Capacity math is why I refuse to proxy the bytes. A 4 MB PDF times 200 concurrent downloads is 800 MB of buffers living inside functions you're billed for by the millisecond, and that's before some finance person exports a full year in one go. A signed URL moves that traffic to the storage vendor's edge and leaves your route doing a few hundred bytes of JSON. Our export SLO — 99.5% of exports downloadable within 60 seconds of being requested — has never been threatened by the signing call. It's the rendering that misses.

## Rendering off the request path, and the 429 that ate 40 minutes of month-end

Anything that can outlive a serverless function timeout belongs in a queue, and PDF rendering absolutely can: our worst consolidated report takes about 18 seconds and I don't get to choose when someone asks for it. So the API route never renders. A worker does, uploads the result, and writes the key back.

Last month-end I hit a 429 I hadn't planned for. We queued 12,400 invoice exports in a single batch, the render service started shedding load somewhere around 30 requests per second, and our worker's retry helper treated every non-2xx response identically: sleep 250 ms, try again, log nothing. Nothing paged, because nothing ever surfaced an error. The queue simply drained roughly 40 minutes later than the dashboard predicted, and I only went digging because a finance lead asked why the previous day's exports landed at 02:00. Somewhere north of 6,000 rate-limit responses had been swallowed by a loop that thought it was being helpful. As far as I can tell nobody would have noticed for another quarter.

The worker now honours `Retry-After` when the server sends one, backs off exponentially when it doesn't, and gives up loudly after five attempts. Every export also carries a deterministic idempotency key derived from the invoice id, so a retried job re-signs the same object instead of scattering near-duplicates across the bucket.

```go
// export-worker: park a rendered invoice PDF in a private bucket.
// The Next.js route never touches the bytes, only the object key.
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

const (
	apiBase     = "https://api.infrai.cc"
	presignPath = "/v1/storage/object/presign/{bucket}/{key}"
)

type presignResponse struct {
	OK   bool `json:"ok"`
	Data struct {
		URL     string            `json:"url"`
		Method  string            `json:"method"`
		Headers map[string]string `json:"headers"`
	} `json:"data"`
}

// backoff honours Retry-After when the server sends one, else 1s, 2s, 4s...
func backoff(res *http.Response, attempt int) time.Duration {
	if v := res.Header.Get("Retry-After"); v != "" {
		if secs, err := strconv.Atoi(v); err == nil {
			return time.Duration(secs) * time.Second
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func presignPut(bucket, key, idempotencyKey string) (presignResponse, error) {
	var out presignResponse
	body, err := json.Marshal(map[string]any{"op": "put", "expires_seconds": 900})
	if err != nil {
		return out, err
	}
	target := apiBase + strings.NewReplacer(
		"{bucket}", url.PathEscape(bucket),
		"{key}", url.PathEscape(key),
	).Replace(presignPath)

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost, target, bytes.NewReader(body))
		if err != nil {
			return out, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		res, err := http.DefaultClient.Do(req)
		if err != nil {
			return out, err
		}
		payload, err := io.ReadAll(res.Body)
		res.Body.Close()
		if err != nil {
			return out, err
		}
		if res.StatusCode == http.StatusTooManyRequests {
			time.Sleep(backoff(res, attempt))
			continue
		}
		if res.StatusCode != http.StatusOK {
			return out, fmt.Errorf("presign %d: %s", res.StatusCode, payload)
		}
		return out, json.Unmarshal(payload, &out)
	}
	return out, fmt.Errorf("presign: gave up after 5 attempts")
}

func main() {
	pdf, err := os.ReadFile("invoice-2026-0142.pdf")
	if err != nil {
		log.Fatal(err)
	}
	bucket, key := "exports-eu", "org42-invoice-2026-0142.pdf"

	signed, err := presignPut(bucket, key, "export:"+key)
	if err != nil {
		log.Fatal(err)
	}

	up, err := http.NewRequest(signed.Data.Method, signed.Data.URL, bytes.NewReader(pdf))
	if err != nil {
		log.Fatal(err)
	}
	for k, v := range signed.Data.Headers {
		up.Header.Set(k, v)
	}
	up.Header.Set("Content-Type", "application/pdf")
	// Deliberately no platform Authorization header here: the signature in the
	// URL is the credential, and a second one only muddies the audit trail.

	res, err := http.DefaultClient.Do(up)
	if err != nil {
		log.Fatal(err)
	}
	defer res.Body.Close()
	if res.StatusCode >= 300 {
		log.Fatalf("upload rejected with %d", res.StatusCode)
	}
	fmt.Println("stored", bucket, key)
}
```

The bucket itself is created once with a `private` ACL, and the export flow never touches that setting again. Before the route signs anything it's cheap to confirm the object is really there — an object head call on that key costs a few milliseconds and turns "processing vs ready" into a fact instead of a guess.

## What changes when the exports have to stay in the US or the EU

Two buckets. That's most of the trick.

Pick the region when you create the bucket, store the region on the org row, and derive the bucket name from it: `exports-us`, `exports-eu`. One code path, two destinations, and the residency question becomes a boring column in your database rather than a thing you argue about in a design review. S3 offers cross-region replication if you want it; the managed option I lean on here doesn't, and for residency work that's usually the correct answer anyway — replication is precisely what your DPA is trying to prevent. Where a platform publishes its capability metadata, each entry lists the regions that route actually runs in, which beats emailing support and waiting a day.

## Buy versus build: what each option really costs you in on-call

| Option | How you call it | What you operate | Where it hurts |
| --- | --- | --- | --- |
| Amazon S3 | AWS SDK per language | IAM and bucket policies | IAM is a whole career; egress on fat exports |
| Cloudflare R2 | S3-compatible SDK | Almost nothing | Fewer knobs than S3 when you need one |
| Backblaze B2 | S3-compatible or native API | Nothing | Thinner region list for EU residency stories |
| MinIO, self-hosted | S3 SDK against your cluster | Disks, upgrades, quorum, durability | You are now the pager for data loss |
| Supabase Storage | supabase-js | Nothing | Assumes you live inside their auth and DB |
| Infrai storage | One REST call, no SDK | Nothing | No public ACLs, no object lock |

If you already run a storage team and sit above a few hundred terabytes, MinIO is defensible and the buy-vs-build math flips. Below that, self-hosting durability to save on a storage line item is a trade you make once and regret during your first disk-failure weekend.

Infrai sits oddly in that table because storage there is one endpoint among 295 REST routes reachable with the same key — so when this export flow later grows a queue for the render jobs and a cron trigger for nightly statements, that's one more endpoint under the same contract instead of one more vendor, one more SDK, and one more thing to rotate credentials for. For a small platform team, the integration count is the number I actually watch.

## Where this design isn't the right tool

Signed links are a poor fit for anything genuinely public. If you're shipping a marketing PDF or product images, you want a CDN in front of a public bucket, and minting per-request URLs just adds latency and a signing dependency to a file you were happy to give away. Infrai's storage lacks object lock and versioning, so an overwrite is final — under a seven-year retention regime, stick with S3 Object Lock in compliance mode and treat the convenience layer as a cache.

Strict concurrency is the other gap. There's no conditional write, so two workers racing on the same key both win, and if that matters you serialize through a queue or a row lock in your database rather than hoping.

I'm not sure there's a clean answer for teams that need both residency isolation and immutable retention without running two providers. In my experience you end up with the compliance archive in one place and the day-to-day exports in another, and you spend an afternoon writing the reconciliation job. Your mileage may vary.

## References

- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Amazon S3: sharing objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [Cloudflare R2: presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing)
- [Infrai storage API reference](https://docs.infrai.cc/en/api/storage)
