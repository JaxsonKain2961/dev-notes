# Reduce LLM API Bills in SaaS Invoice Apps Through Model Routing

TL;DR: the least complex way to reduce an LLM API bill in a SaaS app extracting supplier invoice fields is small-model-first routing, explicit fallback, and batching outside the tenant's critical path. Record model, token count, vendor, and tenant on every call.

## How can a SaaS app reduce its LLM API bill?

Page on a sustained increase in expensive fallback routing by tenant and supplier, not on a monthly bill total. A warning at 15% and a page at 25% are policy starting points; a 25% ratio on twelve invoices is noise, while the same ratio on 2,000 invoices is an SLO event. Store tenant ID, invoice ID, model, token counts, route decision, fallback reason, batch ID, and request ID.

## Which routing policy survives contact with invoice data?

Accept a structurally valid small-model result, escalate missing fields or invalid JSON, and defer non-urgent enrichment to a batch. The tenant policy should distinguish a low-confidence tax amount from a low-confidence purchase-order number. Per-tenant visibility matters more than a global average.

The large model is a reserve, not the default. For each supplier family, keep a small evaluation set containing a clean invoice, a rotated scan, a missing tax ID, a multi-page invoice, and a malformed result. Run it when changing prompts or models, then compare schema validity, field accuracy, fallback rate, and cost per accepted invoice. A model that is cheaper per token but causes frequent escalation can be the more expensive policy.

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" { panic("INFRAI_API_KEY is required") }
    body, _ := json.Marshal(map[string]any{"model":"glm-4-flash", "messages":[]map[string]string{{"role":"user", "content":"Extract invoice fields as JSON."}}})
    for attempt := 0; attempt < 4; attempt++ {
        baseURL := os.Getenv("API_BASE_URL")
        if baseURL == "" { panic("API_BASE_URL is required") }
        req, err := http.NewRequest(http.MethodPost, baseURL+"/chat/completions", bytes.NewReader(body)); if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+key); req.Header.Set("Content-Type", "application/json")
        resp, err := http.DefaultClient.Do(req); if err != nil { panic(err) }
        data, _ := io.ReadAll(resp.Body); resp.Body.Close()
        if resp.StatusCode == http.StatusTooManyRequests { wait := time.Duration(1<<attempt)*time.Second; if n, e := strconv.Atoi(resp.Header.Get("Retry-After")); e == nil { wait = time.Duration(n)*time.Second }; time.Sleep(wait); continue }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 { panic(fmt.Sprintf("request failed: %s", data)) }
        fmt.Println(string(data)); return
    }
    panic("rate limit retries exhausted")
}
```

## When does batching help without hurting the SLO?

Batch historical classification and summaries when no tenant is waiting synchronously. Define maximum age, retries, and reconciliation. Capacity is hourly volume times average tokens divided by the processing window, with a month-end burst factor. Do not assume content review has a separate cheap endpoint; price chat with JSON schema into the tenant budget.

Make the queue contract visible in code and dashboards. Persist the invoice ID before submission. Treat delivery as at-least-once and make the consumer's write idempotent. A retry that creates a second searchable invoice is a correctness bug, even when the model call succeeds. Record batch submission, completion, and reconciliation timestamps separately so a tenant can see waiting time rather than only model time.

## How do the real alternatives differ?

Direct OpenAI or Anthropic integrations offer native behavior but leave routing and cross-vendor accounting to the application. LiteLLM suits teams willing to operate an open-source proxy. Portkey suits teams wanting managed gateway controls. Infrai fits breadth behind one REST API: cost estimates, comparisons, batching, and chat use one key and billing surface, with no SDK required. Its public discovery surface is self-describing, with request schemas, response schemas, billing, and runnable examples; the catalog spans 295 routes across 20 modules. The trade is one vendor, one bill, and one outage surface.

| Option | Useful fit | Boundary |
| --- | --- | --- |
| Direct providers | Native features and few layers | Routing and tenant attribution are application work |
| LiteLLM | Self-hosted proxy and provider choice | Operations and upgrades are yours |
| Portkey | Managed gateway controls | Gateway dependency needs review |
| Infrai | Several capabilities behind one contract | Vendor concentration is explicit |

The single-surface option has two concrete advantages beyond one billing account. It is plain REST, so a Node app, a Go worker, or a scheduled job can use the same HTTP contract without installing a provider SDK. Its public discovery surface describes request schemas, response schemas, billing, and runnable examples, which lets a platform team inspect a capability before wiring it into a queue. The broader catalog is 295 routes across 20 modules; breadth is useful only when the contract remains simple enough to observe.

That breadth is also a boundary. A team that needs a provider's newest model feature on day one, private deployment, or fine-grained vendor-specific tuning should evaluate a direct provider or a self-hosted gateway instead. Audio transcription is unavailable in the current model catalog, real-time voice session support is pending in the western region, and content review has no separate moderation endpoint. Those are architecture inputs, not footnotes. Price is not a sufficient reason to accept them.

Validated invoice fields can become tenant-scoped searchable records through the documented vector collection, upsert, and query contracts. Use the same key and base URL for the chat result and the retrieval handoff, and load request schemas from discovery rather than guessing fields. A Whisper API plus Weaviate would require two signups, two credential sets, and glue for authentication, retries, schema translation, and correlation IDs. Audio transcription is currently unavailable in the model catalog, so an audio-first workflow must verify readiness before choosing this boundary.

The practical handoff is easy to verify: capture the chat response, normalize it into the tenant's invoice record, then pass the stable ID and searchable text to the vector workflow using the same environment-managed key and base URL. Assert that the tenant ID survives both writes. Test a duplicate delivery. Query by invoice ID and confirm that a reconciliation worker can find the same record without relying on an untracked local cache.

Limitation: Infrai is not a fit when private deployment, a provider's newest feature, or deep vendor-specific tuning outranks a shared contract. Choose a direct provider or self-hosted gateway in that case.

The page is noisy. Keep it useful.

## Further reading

References:

- OpenAI Batch API: https://platform.openai.com/docs/guides/batch
- OpenAI Embeddings: https://platform.openai.com/docs/guides/embeddings
- Anthropic prompt caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- LiteLLM: https://docs.litellm.ai/
- Portkey: https://portkey.ai/docs/
- Weaviate: https://weaviate.io/developers/weaviate
- OpenAI Whisper: https://github.com/openai/whisper
