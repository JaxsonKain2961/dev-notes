# Vector Search API Delete by ID: GDPR Erasure Proof for Health PDFs

**TL;DR:** Choose a vector search API only after a destructive test proves that one stable source ID can remove every derived chunk, stale replica, cache entry, and queued reindex job from a health-document question-answering path. Treat `delete by ID` as the trigger, not the evidence of erasure. Keep identity and policy in a separate control plane, stop retrieval before deleting, and issue a receipt only after reconciliation returns zero live artifacts.

A folder of patient PDFs makes the boundary easy to miss. One PDF may become 80 chunks, those chunks may sit in more than one serving copy, and an embedding retry may still be waiting in a queue. Retrieval-augmented generation joins retrieved evidence to generation [1], so an erased source that remains retrievable can still influence a later answer. The deciding constraint is not the delete method's syntax. It is the maximum interval during which any artifact can remain searchable, plus the system's ability to prove what happened.

## How should a vector search API delete data for GDPR erasure?

The right to erasure has exceptions and is not an unconditional purge button; Article 17 defines both the right and the grounds on which continued processing may be required [2]. Have the privacy or records policy decide whether a request is eligible. The retrieval service should enforce the resulting decision, not reinterpret it.

Give each source PDF an opaque `source_id` that survives filename changes and re-uploads. Give every derived object its own `artifact_id`, while retaining `source_id`, a content-version identifier, and an ingestion generation in filterable metadata. Do not use a person's name, email address, or medical-record number as a vector key. The control plane needs one inventory row per source and a manifest of derivations: extracted pages, chunks, embeddings, lexical-index documents, answer-cache keys, and pending jobs.

This is the failure mode to design around: deleting the PDF object does not delete chunks; deleting the chunks does not cancel a delayed worker; canceling current work does not invalidate an old answer cache. A database acknowledgment can be accurate about one subsystem while the user-visible retrieval path remains wrong.

That distinction matters.

The safe order is deliberately conservative:

1. Validate the request and resolve it to one or more opaque source IDs.
2. Mark each source `erasure_pending` in the authoritative inventory. The query path must exclude that state immediately.
3. Cancel or fence ingestion work by generation, so an older worker cannot recreate deleted artifacts.
4. Delete derived records from every serving index and cache, then remove the source object according to the applicable retention decision.
5. Reconcile by source ID across those stores. Emit a completion receipt only when all required checks return zero; otherwise retry from the manifest and alert on the erasure SLO.

Privacy invalidation outranks normal document freshness. A newly uploaded clinical policy may tolerate an indexing delay chosen by the product team, but an accepted erasure must become non-retrievable at the control-plane transition, before physical cleanup finishes.

## Make the identifier contract boring

The API boundary needs three properties: idempotent deletion, metadata-filtered reconciliation by `source_id`, and a read-after-delete verification path whose consistency behavior is documented. A bulk delete operation is useful, but its limitations matter: it is insufficient if it cannot report partial failure or if the only verification is another eventually consistent query with no stated bound.

Here is a small orchestration boundary. The code does not assume that the vector store owns the source of truth, and the generation fence prevents a worker that started before the request from writing after it. Production implementations also need durable retry state and authenticated audit events; those concerns are outside this focused example.

```go
package erasure

import (
	"context"
	"errors"
)

type Manifest struct {
	SourceID    string
	Generation  uint64
	ArtifactIDs []string
}

type Inventory interface {
	BeginErasure(context.Context, string) (Manifest, error)
	CompleteErasure(context.Context, string, uint64) error
}

type Index interface {
	DeleteArtifacts(context.Context, []string) error
	CountBySource(context.Context, string) (int, error)
}

type Jobs interface {
	Fence(context.Context, string, uint64) error
}

type Cache interface {
	PurgeSource(context.Context, string) error
}

type Service struct {
	Inventory Inventory
	Index     Index
	Jobs      Jobs
	Cache     Cache
}

func (s Service) Erase(ctx context.Context, sourceID string) error {
	m, err := s.Inventory.BeginErasure(ctx, sourceID)
	if err != nil {
		return err
	}
	if err := s.Jobs.Fence(ctx, sourceID, m.Generation); err != nil {
		return err
	}
	if err := s.Index.DeleteArtifacts(ctx, m.ArtifactIDs); err != nil {
		return err
	}
	if err := s.Cache.PurgeSource(ctx, sourceID); err != nil {
		return err
	}

	remaining, err := s.Index.CountBySource(ctx, sourceID)
	if err != nil {
		return err
	}
	if remaining != 0 {
		return errors.New("erasure reconciliation found live artifacts")
	}
	return s.Inventory.CompleteErasure(ctx, sourceID, m.Generation)
}
```

The interface never equates a successful delete call with completed erasure. Keep the inventory record as long as the applicable audit and retention policy permits, but keep personal content out of the receipt. A useful receipt carries opaque request and source IDs, the policy decision, timestamps, generation, targeted stores, per-store outcomes, and final status.

## Chunking changes the erasure blast radius

Chunk size is normally discussed as a retrieval-quality choice. In a PDF corpus it is also a deletion and capacity variable. Page-level chunks simplify citations and bounded regeneration, but tables that span pages may lose context. Larger semantic sections preserve more context while increasing the amount of unrelated text returned with a match. Overlapping windows can improve continuity, yet every overlap creates another derived copy that must be inventoried and removed.

Capacity planning starts with amplification, not raw PDF count. For `D` active documents, an average of `C` chunks per document, `R` independently searchable copies, and an overlap or enrichment multiplier `A`, plan for roughly `D x C x R x A` searchable records. This is a planning model, not a benchmark. Measure `C` and `A` from the actual health-document set, especially scanned PDFs and tables, before setting an erasure completion objective.

A manifest is worth its write cost when exact artifact deletion is fast and predictable. Metadata-filter deletion reduces manifest size, but transfers confidence to filtering semantics and index convergence. Full namespace replacement is operationally simple for a small, isolated tenant and expensive for a shared or large corpus. The choice belongs in a buy-versus-build review:

| Decision | Managed index | Self-operated index | Acceptance evidence |
|---|---|---|---|
| Deletion semantics | Contract defines behavior and consistency | Team owns implementation and upgrades | Destructive test over all searchable copies |
| Reconciliation | Requires adequate filters or enumeration | Can inspect storage and index internals | Zero-count query plus inventory comparison |
| On-call load | Less infrastructure, external escalation dependency | More control, larger operational surface | Named owner and timed failure drill |
| Portability | Metadata and filter behavior may constrain migration | Schema is controlled locally | Export-and-delete test using stable IDs |

No row picks a winner. A managed service can lower routine operational work while leaving the team dependent on its deletion and consistency contract; a self-operated system exposes more evidence while making compaction, replication, backups, and upgrades part of the on-call burden. Choose the failure ownership the team can sustain.

There is no free tier of operational responsibility.

## Verification is the feature

Build an acceptance test that plants a uniquely identifiable synthetic PDF, waits until every expected chunk is retrievable, submits erasure, and probes each retrieval path until the agreed deadline. Test the lexical index and answer cache as well as vector similarity. Then release a delayed ingestion message from the old generation and confirm that fencing rejects it.

Test restoration too. Backups may be retained under a lawful policy, but restored data must not silently make an erased source searchable again. Maintain a deletion ledger or equivalent suppression set outside the restored snapshot, replay it before opening query traffic, and verify the source ID remains absent. Article 32 calls for measures appropriate to risk and includes the ability to restore availability and access after an incident [3]; restoration therefore belongs in the erasure threat model.

Four signals expose most gaps: accepted erasures by state, oldest pending request age, reconciliation mismatches by store, and rejected stale-generation writes. Set the SLO from the legal and policy deadline inward, reserving time for retries and human review. Article 12 requires information on action taken without undue delay and generally within one month, with defined conditions for extension [4]; that outer response period is a poor internal cleanup target. The operational objective should be materially shorter and justified by measured queue, index-convergence, and restore-replay behavior.

Keep failure states explicit. `pending` means query suppression is active, `deleting` means cleanup is underway, `verification_failed` means the source stays suppressed while an operator investigates, and `complete` means every required store passed reconciliation. Never roll back by making content searchable again. Roll back a broken deployment to the previous erasure worker, retain the suppression marker, and replay the idempotent job.

## Ship only after the rollback drill

Before production, prove that duplicate requests converge on one outcome, partial deletion resumes without recreating content, timeouts remain ambiguous rather than becoming false success, and a restored snapshot respects prior erasures. Also verify authorization on both request and receipt access; an erasure endpoint is a destructive administrative surface, while a receipt can reveal that a source once existed.

The selection rule is short: reject any vector search API whose deletion boundary, consistency window, filtering behavior, or verification path cannot support the end-to-end erasure SLO. Among the remaining choices, compare measured chunk amplification, on-call ownership, migration effort, and the cost of retaining enough metadata to reconcile.

Delete is an operation. Proof is the deliverable.

## References

1. https://arxiv.org/abs/2005.11401
2. https://eur-lex.europa.eu/eli/reg/2016/679/oj
3. https://eur-lex.europa.eu/eli/reg/2016/679/art_32/oj
4. https://eur-lex.europa.eu/eli/reg/2016/679/art_12/oj
5. https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/individual-rights/right-to-erasure/
