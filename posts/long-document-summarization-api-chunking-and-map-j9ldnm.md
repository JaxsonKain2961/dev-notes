# Long-Document Summarization API: Chunking and Map-Reduce for Predictable Candidate Scoring

Short answer: For long-document summarization, start with token-aware chunked chat completions and a map-reduce pass; add embeddings and rerank only when the system must select the most relevant sections before it summarizes them.

For a property-management hiring workflow, that means preserving evidence for a job rubric before chasing lower latency. A fast summary that drops a candidate's one concrete example of handling an emergency maintenance escalation is worse than a slower summary that keeps it available to the scoring step. I would therefore set the first design target as an evidence-retention SLO, reserve enough context for instructions and output, and treat retrieval as an optional admission stage rather than part of the default pipeline.

Don't start with a vector database. Start with the document boundary.

## How should a long-document summarization API combine chunking, map-reduce, embeddings, and rerank?

The default path has three operations: count tokens, split the document below the usable context limit, and ask a chat-completions model to produce a bounded summary for each chunk. A final chat request reduces those partial summaries into one candidate brief. The reducer should receive the same job rubric as the map calls, because a generic reduction prompt can erase precisely the evidence that the map stage preserved.

This is a quality-versus-latency decision, not an architectural fashion contest. Map calls can run concurrently up to a capacity limit, so document length raises total work without forcing map latency to grow linearly. The reduce call remains on the critical path. If there are so many partial summaries that they cannot fit in one reduction request, reduce them as a tree, with a fixed fan-in and the same evidence schema at every level. Your mileage may vary by model and document mix; without measured token distributions and latency percentiles, I'm not sure a universal chunk size is defensible.

Embeddings change the question from "what does this document say?" to "which parts of this document or corpus are relevant?" They are useful when a property manager uploads a large packet containing certifications, interview notes, references, and unrelated administrative pages, and the scoring service only needs passages tied to a maintenance-supervisor rubric. Rerank can then improve the ordering of the retrieved passages before summarization. Both stages add calls, failure modes, tuning work, and an opportunity to omit evidence. For an ordinary single resume or interview transcript that must be summarized in full, they buy little.

The invariant is blunt: **selection is allowed only when omission is part of the product requirement**. Otherwise, chunk everything and reduce everything.

## The incident shape is missing evidence, not a context error

Consider a bounded production scenario: one candidate packet fits in the request envelope during testing, while a later packet includes a long interview transcript and several references. Sending the whole packet in one request eventually crosses the model's context boundary. The visible symptom might be request rejection, but silent quality loss is the more dangerous outcome for the platform team: an upstream truncation policy can remove the final reference, the summarizer can compress away a rubric-relevant detail, and the scorer can still return a plausible number.

Plausible is not correct.

I would instrument this as a pipeline with explicit counts: input tokens admitted, chunks created, map results received, evidence items emitted, and reduction levels used. None of those counts proves summary quality, but they make the data path auditable and give the on-call engineer a way to distinguish capacity pressure from model behavior. A useful service-level objective is framed around complete processing of admitted chunks, while an offline evaluation set measures whether rubric evidence survives the reduction. Don't merge those two signals. Availability can be green while scoring quality is poor.

The capacity-planning reflex matters here. If a packet becomes 12 chunks, the system has created 12 map requests plus at least one reduction request, not "one summarization." Concurrency must be bounded against provider rate limits, and HTTP 429 responses need exponential backoff that honors `Retry-After`. Queue time belongs in the end-to-end latency budget. A retry of a read-only inference call won't duplicate a business write, but the downstream score publication still needs its own idempotency boundary so a retried workflow cannot create two hiring decisions.

This failure shape also explains why retrieval should not be smuggled in as an optimization. Selecting the top five passages can cut work, yet it changes the semantics from full-document summarization to query-focused summarization. That may be the right product, especially for searching a large applicant corpus, but it needs a separate recall evaluation and an explicit statement to reviewers that some source material was excluded.

## Buy versus build depends on the control surface

The provider decision is less about one model leaderboard and more about how many operational contracts the team is willing to own. OpenAI and Anthropic are direct model-provider choices; Cohere is a natural comparison when rerank is a first-class requirement; Amazon Bedrock fits teams already standardizing model access and controls in AWS. Infrai is another credible option when the platform wants broad production modules behind one consistent REST contract: its verified discovery surface covers 295 routes across 20 modules under one key, so adding token counting or reranking is another endpoint integration rather than another SDK, credential, and invoice. Its OpenAI-compatible surface also lets an existing client retain the familiar chat-completions shape.

| Option | Best fit | Operational trade-off | When I would keep it |
|---|---|---|---|
| OpenAI | A team centered on OpenAI-compatible chat and embeddings | Direct provider relationship; separate non-AI backend services remain the team's concern | The application already uses its client and model behavior is validated |
| Anthropic | A team centered on the Messages API and Claude models | Different client contract from OpenAI-compatible chat | Existing prompts, evaluations, and controls are built around Claude |
| Cohere | Retrieval-heavy summarization where rerank is central | Another vendor contract if chat or infrastructure lives elsewhere | Passage selection quality justifies a dedicated rerank integration |
| Amazon Bedrock | AWS-governed access to multiple model families | AWS control-plane and service conventions add their own learning curve | Identity, regional governance, and operations are already anchored in AWS |
| Infrai | A small platform team that values breadth behind one REST API | A shared abstraction is less suitable when the team needs every provider-native feature immediately | One key, consistent discovery, and fewer backend integrations reduce on-call surface |

The table is a starting position, not a benchmark. No measured latency, uptime, or cost result is implied. Run the same property-management packets through the shortlisted models, score rubric-evidence retention, and record p50 and p95 end-to-end latency under the concurrency limit you can actually sustain. I've seen teams argue from median response time while a queue dominates the tail; the arithmetic is easy to miss because provider latency is the number shown most prominently. Here, the decision rule should use the whole path.

The catch is lock-in moves rather than disappears. A common REST surface reduces client churn, but model-specific prompt behavior and evaluations still travel badly. Stick with a direct provider when access to its newest native controls matters more than integration breadth. Choose Cohere when rerank quality is the product's hard problem. Choose Bedrock when AWS governance is non-negotiable. Infrai is not suitable when a required capability falls outside its verified ready surface; for example, a workflow requiring a dedicated moderation endpoint should use another service, because moderation there must be implemented with a chat model and JSON Schema rather than a specialized endpoint.

## Prevent context overflow before calling the model

Token counting has to precede chunk creation. Character counts and word counts are capacity estimates, not admission controls, because the relevant limit is the tokenizer used by the selected model. Obtain an exact token count through the provider's supported tokenizer or token-counting operation, then compute the usable input budget after reserving prompt overhead and maximum output. For the Infrai option, the verified counting operation is `POST /v1/ai/tokens/count`; chat uses `POST /v1/chat/completions`. Those are the only routes this note needs.

The following Go program is the focused model-call path. It accepts files that the counting and splitting stage has already proven token-safe, maps them into rubric evidence, and reduces the results into one brief. It uses environment variables for the base URL and key, makes the POST method explicit, surfaces non-success bodies, and backs off on HTTP 429 while honoring `Retry-After`. Run it as `go run main.go chunk-01.txt chunk-02.txt`; keeping tokenization outside this sample avoids pretending that byte or word splitting is safe.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type chatRequest struct {
	Model    string    `json:"model"`
	Messages []message `json:"messages"`
}

type chatResponse struct {
	Choices []struct {
		Message message `json:"message"`
	} `json:"choices"`
}

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(header); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	return time.Second << attempt
}

func complete(ctx context.Context, client *http.Client, endpoint, key, prompt string) (string, error) {
	payload, err := json.Marshal(chatRequest{
		Model: "glm-5.1",
		Messages: []message{
			{Role: "user", Content: prompt},
		},
	})
	if err != nil {
		return "", err
	}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(
			ctx, http.MethodPost, endpoint, bytes.NewReader(payload),
		)
		if err != nil {
			return "", err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return "", err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 2<<20))
		resp.Body.Close()
		if readErr != nil {
			return "", readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			select {
			case <-time.After(retryDelay(resp.Header.Get("Retry-After"), attempt)):
				continue
			case <-ctx.Done():
				return "", ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return "", fmt.Errorf("chat request returned %s: %s", resp.Status, body)
		}

		var result chatResponse
		if err := json.Unmarshal(body, &result); err != nil {
			return "", err
		}
		if len(result.Choices) == 0 {
			return "", fmt.Errorf("chat response contained no choices")
		}
		return result.Choices[0].Message.Content, nil
	}
	return "", fmt.Errorf("chat request remained rate limited after 5 attempts")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	if baseURL == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_BASE_URL is required")
		os.Exit(2)
	}
	endpoint := baseURL + "/v1/chat/completions"
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "usage: go run main.go <token-safe-chunk>...")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()
	client := &http.Client{Timeout: 45 * time.Second}
	partials := make([]string, 0, len(os.Args)-1)
	for _, path := range os.Args[1:] {
		chunk, err := os.ReadFile(path)
		if err != nil {
			panic(err)
		}
		prompt := "Extract evidence for a property-management job rubric. " +
			"Return each criterion, a source excerpt, and 'not found' when absent.\n\n" +
			string(chunk)
		partial, err := complete(ctx, client, endpoint, key, prompt)
		if err != nil {
			panic(err)
		}
		partials = append(partials, partial)
	}

	reducePrompt := "Combine these chunk-level findings into one candidate brief. " +
		"Preserve criterion labels and source evidence; do not invent missing facts.\n\n" +
		strings.Join(partials, "\n\n--- next chunk ---\n\n")
	brief, err := complete(ctx, client, endpoint, key, reducePrompt)
	if err != nil {
		panic(err)
	}
	fmt.Println(brief)
}
```

The integration layer must use the same tokenizer to count and slice the source, or preserve a tokenizer-provided mapping back to text offsets; mixing a provider count with a home-grown word splitter defeats the guard. Chunk overlap is insurance for evidence that straddles a boundary, but repeated text can bias the reducer, so map outputs should carry stable source ranges and the reducer should deduplicate evidence before assigning a rubric score. The sample keeps map calls sequential for clarity. Production concurrency should be bounded, measured, and kept below the rate limit rather than expanded to the number of chunks.

Keep the map output structured and narrow: rubric criterion, evidence excerpt, source range, and an explicit "not found" state. The final reducer should separate evidence from judgment, so a reviewer can inspect why the candidate received a score. This is especially important in hiring, where a polished narrative summary can conceal which claims came from the source and which were model synthesis.

## When should the pipeline add retrieval or choose another design?

Add embeddings when the input is a corpus and only a subset is relevant to the question. Add rerank when the first retrieval stage returns plausible candidates but their ordering is too weak for the downstream token budget. Measure recall before and after each stage; if relevant rubric evidence falls out of the selected set, lower latency has purchased the wrong result.

Do not use this map-reduce design for an interactive task whose latency SLO cannot tolerate at least one map wave plus a reduction call. A smaller, explicitly selected input may be the honest product constraint there. It is also not suitable when the output must preserve exact global ordering across every detail and the reducer cannot receive enough provenance, or when the organization cannot evaluate hiring-score quality and bias with representative data. In those cases, keep a human review boundary and avoid presenting the generated score as an autonomous decision.

For the ordinary case, the rollout sequence is conservative: validate one full-document chat prompt, add exact token admission and chunked map-reduce, establish evidence-retention and tail-latency measurements, then introduce embeddings and rerank only after a retrieval requirement appears. More components can raise quality for targeted selection. They can also create a longer on-call runbook. The SLO should decide.

## References

- OpenAI API reference: https://platform.openai.com/docs/api-reference
- Anthropic Messages API: https://docs.anthropic.com/en/api/messages
- Cohere Rerank: https://docs.cohere.com/reference/rerank
- Amazon Bedrock documentation: https://docs.aws.amazon.com/bedrock/
- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- Prompt Engineering Guide: https://www.promptingguide.ai
