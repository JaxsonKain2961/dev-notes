# Batch Audio Transcription for Long Recordings: Webhook Contracts That Survive Retries

Short answer: use an asynchronous batch job with a durable upload, an idempotency key, and a signed webhook that is treated as an event, not as proof that the transcript is already usable. That is the least complex design that remains honest for long support calls and podcasts; a synchronous request is appropriate only when the latency SLO and recording size are both small.

I learned this from a bounded production incident. We submitted a 2-hour-17-minute support recording, received HTTP 200, and closed the intake ticket. Four hours later the transcript was still absent. The request had been accepted, but our worker lease expired before the next state was recorded, and the callback path had no event ledger. Every request-error panel was green. I replayed the gateway trace, checked the object-store access log, compared the provider request identifier with our database row, and then ran a duplicate submission against a 38-minute sample to prove that a retry would not create a second logical job. The useful metric turned out to be the age of the oldest non-terminal job, not the status code on the upload request; that single change also gave the on-call engineer a bounded list of records to inspect instead of an empty error dashboard.

That failure changed the job contract.

An accepted request is not a completed transcription. I persist an application-owned job before making the network call, derive its idempotency key from the recording identity, and require an observed terminal event before publishing the transcript. This sounds fussy until a retry arrives while the first attempt is still writing.

## The invariant: accepted is not complete

The state machine should be deliberately boring: `accepted`, `processing`, `succeeded`, `failed`, and `expired`. Each transition records an event ID, observed time, and source. A duplicate webhook becomes an acknowledged no-op; a missing webhook becomes an age metric that a reconciler can act on.

```go
type Job struct {
	ID             string
	Source         string
	IdempotencyKey string
	State          string
	Attempt        int
}

func acceptTranscription(source string) (Job, error) {
	job := Job{
		ID:             newID(),
		Source:         source,
		IdempotencyKey: hash(source),
		State:          "accepted",
	}
	if err := store(job); err != nil {
		return Job{}, err
	}
	return job, nil
}
```

The submit endpoint returns a job identifier quickly. Audio belongs in durable object storage, with checksum, duration, channel count, and retention class stored beside the job. Passing a reference rather than hours of bytes through an API gateway keeps gateway timeouts out of the critical path. A maximum duration and size are capacity controls: one three-hour podcast should not consume every worker slot.

The webhook handler verifies the signature against the raw body, rejects stale timestamps, and writes the event ID before returning 2xx. If transcript indexing or redaction is slow, the handler enqueues that work and acknowledges the callback; the queue owns retries and dead-letter handling. The callback is never the only record of state.

## What should an async transcription API promise for long recordings, support calls, and podcasts?

It should promise a retry-safe contract, not a particular model score. RFC 9110 distinguishes an idempotent operation from a client action that merely happens to be repeatable. The submission key maps to one logical job, and replaying the same webhook must not create a second transcript version or billable downstream action. Return the same job identity for a repeated key, or return a clear conflict when the key is reused for different audio.

Keep reconciliation in the design even when webhook delivery has a strong service guarantee. A small loop scans the oldest `accepted` and `processing` jobs, asks for status with bounded exponential backoff, and stops after an age limit tied to the business SLO. Your mileage may vary on the interval; rate limits and completion objectives matter more than a fashionable five-minute timer.

Long recordings also need an explicit completion policy. Define what happens when a provider reports `processing` beyond the target window, when the source object is deleted early, and when a transcript is partial. A partial result can be useful for an internal review, but it must not silently replace the immutable final version that search and compliance exports consume.

## Calls and podcasts expose different failure modes

Support calls carry tenant, consent, language, and retention metadata. Podcasts more often need chapter boundaries, speaker turns, and a public-release review. Keep those attributes next to the job, but do not trust client labels for authorization. Treat redaction, diarization, chaptering, and indexing as separate stages with their own queues, budgets, and failure states.

Audio quality is another capacity variable. Measure duration, sample rate, channel count, detected silence, and codec at ingest. A two-channel call with known speakers permits channel-aware processing; a mixed podcast track needs a different expectation. Store original bytes and a normalized derivative so a model change does not force a new upload. Transcript versions should be immutable, with the search index pointing at a selected version.

The incident pattern is predictable: a callback is delivered before the upload transaction commits, a worker dies after writing half a transcript, or a duplicate event races the first write. Test these sequences with fault injection.

A green HTTP panel is not evidence that the workflow is complete.

## Buy, build, or split the boundary?

The decision is about ownership of the on-call surface. A managed batch API can remove model serving and capacity operations, while a self-hosted stack gives tighter control over the data path and scheduling. A hybrid keeps the job contract and event ledger in the application and lets workers change behind that boundary.

| Choice | You own | You gain | You give up |
| --- | --- | --- | --- |
| Managed batch service | storage, submission, callbacks, reconciliation | elastic model operations | provider limits, residency choices, and lock-in |
| Self-hosted speech stack | GPUs, serving, upgrades, quality tuning | control over data path and scheduling | capital, paging load, and slower model turnover |
| Hybrid queue with pluggable workers | routing, contracts, two runtime paths | migration room and workload-specific tuning | more integration tests and duplicated observability |

I favor the hybrid boundary when transcripts are durable product records and the team can operate a queue, object storage, and an on-call rotation. The catch is real engineering ownership; it is not suitable when staffing or compliance review makes that surface the larger risk. Stick with a managed workflow when operational capacity is scarce. Choose self-hosting when predictable volume and strict residency outweigh elasticity.

## SLOs and capacity planning after launch

Measure accepted-to-processing latency, processing-to-transcript latency by audio hour, terminal success rate, and webhook-to-acknowledgment latency. Track queue age, retry count, duplicate event rate, reconciliation repairs, retained bytes, and the percentage of jobs past their completion objective. These are workflow signals, not vanity API latency.

For a first capacity estimate, required worker audio-hours per wall-clock hour equals incoming audio-hours divided by the target completion window, multiplied by headroom for retries and maintenance. Measure real-time factor separately for calls and podcasts, then load-test with long files rather than only ten-second fixtures. Include checksum calculation, object-store reads, diarization, redaction, and indexing in the budget.

Cost is multidimensional: compute, storage, egress, queue operations, observability, and engineer-hours spent on failed jobs. A lower per-minute quote can lose after reprocessing and retention, while a higher quote can buy capacity the team would otherwise page about. I am not sure a universal winner exists; workload shape and contractual limits decide it.

Completion is an observed fact. Design the ledger, retries, and reconciliation around that sentence, and long-form transcription becomes a controlled workflow instead of a hopeful HTTP call.

## Further reading

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.promptingguide.ai
