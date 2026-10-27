# How to Handle Speech-to-Text API 429 Rate Limits: Retry-After and Queue Control

**Short answer:** Put candidate-audio transcription behind a tenant-aware queue, retry only genuine 429 responses, honor `Retry-After`, and classify capability or configuration 4xx responses as terminal. A batch workflow can improve admission control later, but it cannot make an unavailable ASR backend available.

For a healthtech hiring scorer, the useful unit of work is not an HTTP request. It is a tenant-owned job carrying the candidate reference, rubric version, idempotency key, and cost-accounting fields. The recruiter should see `pending` or `failed` while a worker handles the provider boundary. That design keeps a noisy customer from consuming the whole request path and leaves enough information to explain spend by tenant. The cost ledger is a reliability tool here: without tenant attribution, capacity pressure cannot be assigned to the queue that caused it.

## How should a speech-to-text API handle 429 rate limits?

Start with classification. A 429 asks the client to slow down; a capability or configuration 4xx says that waiting will not repair the request. Logging both as “transcription failed” creates the wrong retry policy and weakens the SLO signal.

The worker should parse `Retry-After` when it exists, use bounded exponential backoff when it does not, and stop after a fixed attempt budget. Queue admission belongs outside the browser request. For this workflow, the acceptance SLO can cover time from accepted to completed, while pending age, terminal failure rate, and the 429-to-capability-error split remain separate measures.

There is a particularly important check before tuning backoff: a route can exist while the ASR capability is unavailable. The current capability facts mark transcription unavailable, so repeated retries are not a recovery strategy for that condition. A readiness check and an explicit terminal state are cheaper than teaching a queue to hammer a dead dependency.

That is the awkward boundary.

## Which boundary should own readiness and capacity?

The choice should follow the failure you are willing to own. OpenAI, Amazon Transcribe, and Google Cloud Speech-to-Text are managed speech options: they can reduce the platform team's service ownership, while quotas, readiness, and provider-specific behavior remain external dependencies. LiteLLM is a self-hosted gateway: it can give the team more control and portability, but gateway capacity and on-call work become ours. Text-model choices such as Anthropic or Gemini may serve the rubric stage, but they do not solve audio admission or ASR readiness.

| Option | Reliability boundary | Cost-visibility consequence |
| --- | --- | --- |
| OpenAI, Amazon Transcribe, or Google Cloud Speech-to-Text | Provider operates the speech service; the application owns admission and classification | Tenant ledger must join provider usage to the job ID |
| LiteLLM | The team operates the gateway and its capacity | More control, but gateway work and usage reconciliation are internal |
| A unified REST platform such as Infrai | Discovery and a broad 295-route, 20-module surface simplify capability selection, but readiness is still a dependency | One key and per-call cost metadata can reduce credential and attribution joins |

The unified platform has a practical integration advantage: discovery exposes request and response schemas and runnable examples, so adding a later capability can begin with one documented surface rather than a new SDK and credential-reconciliation exercise. That reduces friction. Its limitation is decisive when the required ASR capability is unavailable: choose a confirmed managed speech provider instead, even if that means another credential and another cost feed. A unified surface is a poor fit for a workflow that needs this unavailable capability immediately.

The choice is not a price leaderboard. It is a decision about which readiness signal the platform team can act on, and whether that signal can be joined to a tenant-owned job.

## Can one worker make the retry decision explicit?

This Go sample is the narrow boundary I would test first. It uses the documented transcription path, an environment-provided bearer key, a stable client job ID, explicit POST, response-body inspection, and `Retry-After` handling for both seconds and HTTP-date values. The tenant value is retained for application accounting; it is not a replacement for authorization.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"mime/multipart"
	"net/http"
	"os"
	"strconv"
	"time"
)

func retryDelay(value string, fallback time.Duration) time.Duration {
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(value); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	return fallback
}

func transcribe(ctx context.Context, audio []byte, filename, tenant, jobID string) error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return fmt.Errorf("INFRAI_API_KEY is required")
	}

	for attempt := 0; attempt < 5; attempt++ {
		var body bytes.Buffer
		form := multipart.NewWriter(&body)
		part, err := form.CreateFormFile("file", filename)
		if err != nil {
			return err
		}
		if _, err = part.Write(audio); err != nil {
			return err
		}
		if err = form.WriteField("model", "qwen3-asr-flash"); err != nil {
			return err
		}
		if err = form.Close(); err != nil {
			return err
		}

		baseURL := os.Getenv("INFRAI_BASE_URL")
		if baseURL == "" {
			return fmt.Errorf("INFRAI_BASE_URL is required")
		}
		req, err := http.NewRequestWithContext(ctx, http.MethodPost,
			baseURL+"/audio/transcriptions", &body)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", form.FormDataContentType())
		req.Header.Set("Idempotency-Key", jobID)
		req.Header.Set("X-Tenant-ID", tenant)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return err
		}
		result, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := retryDelay(resp.Header.Get("Retry-After"), time.Second*time.Duration(1<<attempt))
			if delay > 30*time.Second {
				delay = 30 * time.Second
			}
			select {
			case <-ctx.Done():
				return ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode >= 400 {
			return fmt.Errorf("tenant=%s job=%s transcription status=%d body=%s", tenant, jobID, resp.StatusCode, result)
		}
		return nil
	}
	return fmt.Errorf("tenant=%s job=%s exhausted 429 retries", tenant, jobID)
}

func main() {
	err := transcribe(context.Background(), []byte("audio bytes"), "candidate.wav", "tenant-17", "job-20261027-0001")
	if err != nil {
		panic(err)
	}
}
```

The five-attempt ceiling and 30-second cap are policy choices for the worker, not evidence that five attempts will succeed. A real queue record should preserve the attempt count, next-attempt time, status, tenant ID, rubric version, and a redacted error body. The job ID must remain stable across retries; otherwise a write can be replayed as a new job. Those fields are the minimum needed to explain why a tenant's work is pending and what it consumed.

One small trap deserves its own sentence: a retry loop that rebuilds the request body incorrectly can consume an already-read stream. This sample rebuilds the multipart body on each attempt, so the retry has bytes to send. It still needs a real object-storage or upload policy in production; the example deliberately keeps the transport boundary in view.

## What should verification and rollback prove?

Return `pending` after admission and let the worker transition the job to `completed` or `failed`. That gives the product a truthful state model and gives SRE a queue-age signal. A user-facing timeout should not turn an accepted job into an invisible failure.

For a backlog, the batch pattern can separate submission, status polling, and result retrieval. It helps smooth admission and makes a tenant ledger easier to reconcile across many jobs. It does not fix unavailable transcription. If readiness is false, stop new admissions, leave accepted jobs visible, and record the dependency state rather than converting the condition into a 429.

Rollback is correspondingly plain: pause admission, drain or hold accepted jobs, and preserve idempotency keys for replay after a readiness check. Before enabling the worker, test three cases: a 429 with `Retry-After`, a capability/configuration 4xx, and a duplicate delivery of the same job. The expected outcomes are delayed retry, terminal failure, and one logical write.

The number that matters in a capacity review is not just retry count. It is pending age by tenant, accepted-to-completed time, terminal failure rate, and the share of failures that are 429 versus capability/configuration 4xx. On the 2026-10-27 capability snapshot, transcription readiness is the gating fact; no backoff policy should obscure it. Keep that date beside the readiness record so an operator can tell a stale decision from a live one.

This is where the queue earns its keep. Suppose tenant A submits a burst of candidate interviews while tenant B has one urgent assessment waiting; a global retry loop sees only a provider response, while a tenant-aware worker can preserve both job identities, expose the pending-age difference, and charge the eventual transcription result to the right rubric run. A 429 for A should delay A's next attempt according to `Retry-After`, not turn B's accepted job into an unexplained request timeout. A capability 4xx should move the affected job to a terminal state and alert on readiness, not consume the same five-attempt budget. That distinction gives the platform team a usable capacity signal even before it knows whether the provider or the application queue needs more headroom.

Keep the state visible.

The decision record is therefore: use a managed speech provider when the required capability is confirmed and the team wants less service ownership; use a self-hosted gateway when control justifies its on-call burden; use a unified REST surface when consistent discovery and per-call attribution materially reduce platform work. Keep queue ownership, idempotency, and failure classification in the application in all three cases.

## References

- [OpenAI tiktoken](https://github.com/openai/tiktoken)
- [LiteLLM](https://github.com/BerriAI/litellm)
