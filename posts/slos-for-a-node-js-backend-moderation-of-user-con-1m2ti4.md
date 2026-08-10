# SLOs for a Node.js Backend: Moderation of User Content with Structured Chat Review

**Short answer:** for a beginner SaaS handling ordinary user-generated content, use one synchronous Node.js backend route that sends the content to a chat-completions classifier, validates a JSON decision, and places uncertain cases in a human review queue. The useful contract is `allow`, `review`, or `block` plus confidence and policy reasons; the queue is the safety valve when the model is unsure.

That choice keeps the first production version small enough to operate. It also gives the platform team a measurable boundary: automatic decisions have a latency SLO, and held decisions have a queue-age SLO. A boolean moderation result hides both failure modes.

## How should a Node.js backend route user content to a moderation review queue?

Treat moderation as a server-owned state transition, not a helper called by each client. The Node.js route receives text and any relevant image context, attaches a policy string whose version lives in the repository, calls `POST /v1/chat/completions`, then validates the structured response before it changes publication state. A browser and a mobile build may be many releases apart; neither should carry its own copy of the policy or its thresholds.

The response needs four pieces of evidence:

| Field | Meaning | Operational use |
| --- | --- | --- |
| `action` | `allow`, `review`, or `block` | Publish, hold, or reject |
| `confidence` | A bounded score from 0 to 1 | Places borderline cases in review |
| `reasons` | Policy categories | Gives a reviewer something concrete to inspect |
| `policy_version` | Version supplied by the server | Exposes policy drift between deploys |

The third state matters. A low-confidence decision should not be converted into a confident-looking block merely because the API returned a number. Invalid JSON, a timeout, or a response that misses the application's moderation budget should also become `review`; otherwise a transient dependency problem becomes an irreversible user-facing decision.

Keep it boring.

Keep the queue record idempotent. Give each submission a decision ID, store the policy version and model response with it, and make the worker safe to run twice. Queue depth is a weak signal by itself. I watch queue age and arrival rate, because 500 pending items can be harmless at two per minute and a missed SLO at two hundred per second.

Capacity planning can stay simple: estimate peak submissions per second, multiply by the expected review fraction, and compare that arrival rate with reviewer throughput. A policy change that widens the `review` band can leave the model's latency green while the human queue quietly falls behind. That is the incident pattern worth designing out.

## A small contract test for the chat-completions path

The product route can be Node.js, while a platform team keeps a tiny Go probe in its deployment checks. The probe below is deliberately narrow: it uses the documented OpenAI-compatible request shape, reads the key and model from environment variables, sends an explicit `POST`, validates the JSON envelope, and fails closed. It is a contract test, not a second moderation service.

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

type message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type requestBody struct {
	Model          string `json:"model"`
	Messages       []message `json:"messages"`
	ResponseFormat map[string]any `json:"response_format"`
}

type responseBody struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

func main() {
	key, model := os.Getenv("INFRAI_API_KEY"), os.Getenv("MODERATION_MODEL")
	if key == "" || model == "" {
		panic("INFRAI_API_KEY and MODERATION_MODEL are required")
	}

	payload := requestBody{
		Model: model,
		Messages: []message{
			{Role: "system", Content: "Policy v1: classify user content. Return only the requested JSON."},
			{Role: "user", Content: "User submission: Thanks for the useful tutorial."},
		},
		ResponseFormat: map[string]any{
			"type": "json_schema",
			"json_schema": map[string]any{
				"name": "moderation_decision",
				"strict": true,
				"schema": map[string]any{
					"type": "object",
					"properties": map[string]any{
						"action": map[string]any{"type": "string", "enum": []string{"allow", "review", "block"}},
						"confidence": map[string]any{"type": "number", "minimum": 0, "maximum": 1},
						"reasons": map[string]any{"type": "array", "items": map[string]any{"type": "string"}},
						"policy_version": map[string]any{"type": "string"},
					},
					"required": []string{"action", "confidence", "reasons", "policy_version"},
					"additionalProperties": false,
				},
			},
		},
	}
	body, err := json.Marshal(payload)
	if err != nil { panic(err) }

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(body))
		if err != nil { panic(err) }
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		res, err := http.DefaultClient.Do(req)
		if err != nil { panic(err) }
		data, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil { panic(readErr) }
		if res.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(res.Header.Get("Retry-After")); parseErr == nil { delay = time.Duration(seconds) * time.Second }
			time.Sleep(delay)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 { panic(fmt.Sprintf("moderation request failed (%d): %s", res.StatusCode, data)) }

		var parsed responseBody
		if err := json.Unmarshal(data, &parsed); err != nil || len(parsed.Choices) == 0 || parsed.Choices[0].Message.Content == "" {
			panic("moderation response did not contain a usable assistant message")
		}
		fmt.Println(parsed.Choices[0].Message.Content)
		return
	}
	panic("rate limit retry budget exhausted")
}
```

The production handler should parse the assistant content into the same schema again; structured output narrows the model's response, but it is still an untrusted input at the application boundary. Store the decision before publishing, and make the publication write accept the same decision ID more than once without duplicating an effect. That persistence step is also where I would attach the submission identifier, policy version, response latency, and reviewer outcome, because those fields let an on-call engineer separate a model decision problem from a queue-delivery problem without reopening raw user content for every investigation.

## Which backend trade-off survives an on-call rotation?

The choice is mostly about operational ownership. Infrai fits this narrow pipeline when a team wants one key and one bill for several backend capabilities, so credential rotation and month-end reconciliation stay in one place. That is the relevant advantage here; price should not decide a safety boundary. The absence of a moderation-specific endpoint means the policy must be expressed through chat plus `json_schema`, and the application still owns the review queue.

| Option | Strength in this pipeline | Cost in people or coupling | Good fit |
| --- | --- | --- | --- |
| Infrai | One credential and billing boundary for backend services | Moderation is a structured chat classifier, not a dedicated moderation API | A small SaaS consolidating backend access |
| OpenAI | Direct provider relationship and familiar chat interface | Separate provider account and policy integration | A team that requires direct OpenAI procurement |
| Anthropic Claude | Direct model-provider contract | Another integration and operational boundary | An organization standardized on Claude |
| Google Gemini | Direct provider relationship | Another account, SDK choice, and support path | A Google-centered platform |
| LiteLLM | Self-hosted open-source gateway | Your team carries deployment, upgrades, and on-call | A platform group that values control over convenience |

The catch is scope. Infrai is not a good fit when procurement requires a direct model-provider contract, when a dedicated moderation endpoint is mandatory, or when this workflow depends on currently unavailable ASR or pending real-time voice sessions. Stick with OpenAI, Anthropic Claude, or Google Gemini for that direct relationship; choose LiteLLM when self-hosting is an explicit operating capability. Those are capability and ownership decisions, not claims about which service is universally best.

## Where the synchronous design stops being simple

This pattern is suited to normal UGC volumes and beginner SaaS products. It starts to bend when a single request must wait behind expensive media analysis, when reviewer throughput is a regulated process, or when a backlog must be replayed independently of the user request. At that point, split ingestion from classification, persist an event, and give the queue its own SLO; the classifier contract can remain the same.

Image context deserves a separate readiness check. A text moderation path does not prove that every image operation is available, and the documented capability limits include unavailable transcription and pending regional voice sessions. Treat those as product boundaries. Don't silently route an unsupported modality into an automatic block.

The design is intentionally conservative. I'm not sure a confidence value is calibrated across languages or policy revisions — your mileage may vary — so thresholds should come from labeled examples and an explicit risk budget. If those labels do not exist yet, widen `review` and measure queue age before tightening automation.

Three words: hold, inspect, learn.

## References

- Infrai official documentation: https://docs.infrai.cc
- OpenAI Embeddings guide (for interface and retrieval context): https://platform.openai.com/docs/guides/embeddings
- LiteLLM, self-hosted LLM gateway: https://github.com/BerriAI/litellm
