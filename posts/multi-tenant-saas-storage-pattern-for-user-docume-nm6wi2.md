# Multi-Tenant SaaS Storage Pattern for User Documents: PDF, DOCX, Invoices

Retention is the constraint that should shape a gaming SaaS document store, because deleting an old invoice or a player's uploaded PDF is a distributed workflow, not a row deletion. **Short answer: keep immutable document identity and tenant ownership in database metadata, keep the bytes in a private object store, and make retention a state machine with an explicit deletion ledger.** Folder prefixes help operators find a scope; they must not decide who may read a file or whether a file is safe to delete.

My first check is a deletion deadline, not a bucket layout. If the product says that a tenant's documents are removed after 14 days of inactivity, the system needs to prove which policy version applied, which objects were selected, and when each deletion became observable. It also needs to restore one selected snapshot without quietly resurrecting documents that were already deleted.

That is the operational shape of the problem.

## What should a multi-tenant SaaS storage pattern do with PDF, DOCX, invoices, and private bucket prefixes?

Use a database catalog as the authoritative index and a private bucket as the byte store. A key such as `tenant/{tenantID}/document/{documentID}/content` is useful because it gives reconciliation jobs a bounded tenant scope and avoids filename collisions. It is not an access-control rule. A user-controlled filename should remain in metadata, where it can be displayed or searched without changing the object path.

The catalog row should include a document ID, tenant ID, owner ID, object key, media type, original filename, byte size, creation time, retention policy version, and lifecycle state. The important field is the state, because a document can be present in one system while unavailable in another. A practical sequence is `pending`, `available`, `retention_pending`, `deleting`, `deleted`, and `restore_pending`.

The application authorizes every read against the tenant and owner in that row. It then obtains the object through a short-lived, application-controlled operation rather than exposing a predictable public URL. Cache behavior deserves the same care: private document responses should carry a cache policy appropriate for user-specific data, and a browser should not retain a response longer than the product's authorization model permits. The HTTP `Cache-Control` reference is a useful standard checkpoint here.

Do not make a folder tree your database.

The database answers “which invoice belongs to this tenant and is still available?” with an indexed query. Prefix listing answers “which object names currently exist under this operational scope?” Those are different questions, and confusing them creates an on-call problem that gets worse as retention grows.

## How should retention and deletion work across private bucket storage and database metadata?

Separate eligibility from physical deletion. A scheduled worker can select catalog rows whose policy says they are eligible, record a deletion intent, and move them to `retention_pending`. A second step claims work idempotently, verifies the tenant and object key from the catalog, removes the bytes, and then records `deleted`. If the byte removal is retried, the operation should converge on the same terminal state; the worker must not create a new document ID or infer ownership from a prefix listing.

That split matters for invoices. A business rule may say “delete after 14 days,” while a legal hold, an unresolved chargeback, or an active restore request says “not yet.” Those exceptions belong in queryable metadata with an owner and an expiry or review condition. A retention job that only compares timestamps is fast to write and hard to defend.

Here is a compact Go model for the decision boundary. It does not perform storage operations; it makes the dangerous transition explicit so the worker and its tests cannot accidentally delete an active or held document.

```go
package retention

import "time"

type Document struct {
	TenantID          string
	ID                string
	State             string
	CreatedAt         time.Time
	RetentionUntil    time.Time
	LegalHold         bool
	RestoreInProgress bool
}

func EligibleForDeletion(doc Document, now time.Time) bool {
	if doc.State != "available" || doc.LegalHold || doc.RestoreInProgress {
		return false
	}
	return !now.Before(doc.RetentionUntil)
}
```

The worker should claim by document ID and policy version, not by a path assembled from the latest request parameters. Store a deletion event with the actor (`retention-worker` or an operator), the reason, the policy version, and a timestamp. That event is the evidence needed to explain a missing invoice to support and to distinguish an intentional deletion from an orphaned object.

The catch is that object storage and a relational database do not share one transaction. A successful database update followed by a failed byte deletion leaves an orphan; a byte deletion followed by a failed database update leaves a missing object. Both are normal crash outcomes. Design reconciliation for them instead of pretending a two-system commit exists: rows in `deleting` past their lease are retried, and objects with no matching catalog row are quarantined for review rather than immediately removed. In a gaming SaaS, imagine a tenant with 12,000 invoices and a 14-day retention rule changing at 02:00 UTC while three workers are processing the previous policy version. One worker may have claimed a row, another may have observed the old deadline, and a third may have finished the byte operation before the catalog transaction was committed. The correct response is not to rerun the entire prefix or to delete every object older than the new cutoff. Freeze claims for that policy, preserve the leases and deletion events, compare the catalog's policy version with the worker's claim, and replay only rows whose state makes the transition valid. This costs more database design up front, but it gives the incident commander a bounded set of decisions: retry a leased deletion, quarantine an unmatched object, or create an explicit recovery record. Without that ledger, an operator is forced to infer intent from timestamps and names, which is exactly how a tenant boundary becomes an incident.

## The failure modes that make document retention unsafe

The first failure is an over-broad key prefix. If tenant identity is inferred from a request path or a mutable display name, one authorization mistake can select another tenant's documents. Generate keys from canonical IDs, reject empty or malformed IDs, and never concatenate an untrusted filename into a policy query.

The second is a retention query that has no stable clock or policy version. A worker running during a policy change can process half of a tenant's documents under the old rule and half under the new one. Save the effective policy on each catalog row, record the evaluation time, and make changes explicit. “The current setting says 14 days” is not an audit trail.

The third is deletion that cannot be observed. A count of successful worker loops is not enough. Track eligible rows, claimed rows, deletion attempts, terminal deletions, retry age, orphan candidates, and restore completions, partitioned by tenant scope where the cardinality is safe. The SLO should cover the user-visible result, such as the time from eligibility to catalog state `deleted`, while a separate operational objective covers the age of unclaimed work.

Capacity planning belongs here. If a worker deletes 600 objects per minute and a daily run creates 18,000 eligible objects, the nominal work takes 30 minutes before retries, database contention, and provider throttling. Those figures are planning inputs, not a promise; your mileage may vary when document sizes, rate limits, or tenant distribution change. Keep the queue bounded, use leases, and reserve headroom for a restore event rather than running deletion at the platform's maximum throughput.

Finally, restore can undo a deletion decision without undoing a retention decision. A selected snapshot should be restored into a new document version or a clearly marked recovery state, with the original deletion event retained. Never reuse the deleted row's object key silently. The operator needs to see what was restored, under which authorization, and whether the retention clock starts again.

## A Go runbook for safe selection, verification, and rollback

The runbook is intentionally plain. First, pause new retention claims for the affected tenant or policy version. Second, inspect the catalog, deletion ledger, queue leases, and reconciliation report. Third, resume only after the selection query and the byte-operation result agree for a sampled set.

The selection query should be narrow and repeatable. It should filter by tenant scope, state, retention deadline, legal-hold status, restore status, and policy version. It should claim rows in a transaction with a lease, then commit before doing a slow object operation. A worker crash after the claim is recoverable; a claim without an expiry is an operator-created outage.

```go
package retention

import (
	"context"
	"fmt"
)

type Catalog interface {
	Claim(ctx context.Context, documentID, policyVersion string) (bool, error)
	MarkDeleted(ctx context.Context, documentID string) error
	MarkRetryable(ctx context.Context, documentID string, reason string) error
}

type ByteStore interface {
	Delete(ctx context.Context, objectKey string) error
}

func DeleteOne(ctx context.Context, catalog Catalog, store ByteStore, doc Document, policyVersion string) error {
	claimed, err := catalog.Claim(ctx, doc.ID, policyVersion)
	if err != nil || !claimed {
		return err
	}

	if err := store.Delete(ctx, docKey(doc)); err != nil {
		_ = catalog.MarkRetryable(ctx, doc.ID, fmt.Sprintf("byte deletion deferred: %v", err))
		return err
	}
	return catalog.MarkDeleted(ctx, doc.ID)
}

func docKey(doc Document) string {
	return "tenant/" + doc.TenantID + "/document/" + doc.ID + "/content"
}
```

In production, the catalog should supply the canonical object key instead of reconstructing it, and the worker should validate that the key's tenant segment matches the row. The example keeps that rule visible. It also makes the awkward case visible: a byte deletion can succeed while the final metadata update fails, so reconciliation must find the mismatch and the delete event must make the intended outcome clear.

Verification should include tenant-boundary tests, duplicate delivery tests, policy-change tests, held-invoice tests, partial-failure tests, and restore-after-delete tests. Exercise real object sizes that resemble PDFs, DOCX files, and invoice batches; a test suite containing only tiny fixtures says little about queue pressure. Add a browser upload test with progress reporting when the client owns the upload, but keep authorization and retention decisions on the service side.

Rollback means stopping claims and making pending work retryable. It does not mean recreating deleted bytes from a blind bucket scan. If a restore is required, require an explicit document ID and snapshot ID, write a new catalog record, and preserve the deletion ledger. The operator should be able to answer three questions before reopening the queue: what was selected, what actually disappeared, and what will happen on the next retry?

## Buy versus build: where this pattern stops fitting

The database-plus-private-bucket pattern is a good boundary for ordinary user documents when the product already has a transactional database and needs tenant-scoped authorization, searchable metadata, and controlled deletion. It is a poor fit when the requirement is immutable regulatory retention, built-in content indexing, or a managed records workflow that the team cannot operate itself.

| Choice | It makes sense when | The limitation to accept |
| --- | --- | --- |
| Generic object storage plus a database catalog | The team needs control over keys, metadata, retention policy, and restore semantics. | The team owns reconciliation, worker capacity, audit events, and access controls. |
| A self-hosted storage service | Data placement and operational control outweigh the cost of running storage infrastructure. | On-call load includes capacity, upgrades, durability checks, and recovery exercises. |
| A managed document or records service | Legal hold, retention evidence, and search are the primary product requirements. | Its workflow and data model can constrain migrations, tenant isolation, and custom restore behavior. |

Stick with a managed records system when deletion must be legally provable and the team cannot staff that operation. Choose the simpler catalog pattern when the real requirement is private storage for user PDFs, DOCX files, and invoices with an application-owned lifecycle. Do not choose on storage price alone; the recurring cost is the failure-handling surface and the people required to keep it honest.

I'm not sure any generic architecture can determine the correct retention period for a gaming business. That depends on contracts, jurisdiction, fraud controls, and the meaning of an invoice in the product. Resolve those inputs first, then encode the decision as versioned metadata rather than burying it in a cron expression.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
