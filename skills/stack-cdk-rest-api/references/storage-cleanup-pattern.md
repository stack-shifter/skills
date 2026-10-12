# Storage Cleanup Pattern

Use when implementing uploads, generated exports, or deletion spanning a datastore and S3. Reuse existing tracking and cleanup
infrastructure; the names below describe roles, not mandatory tables.

## Business Deletion

In one database transaction, persist the owed object keys and delete the business rows. After commit, attempt S3 cleanup. Remove owed
entries only for successful object deletions; retain failures for the scheduled worker. A successful HTTP deletion confirms database
removal, not completion of every S3 deletion. Avoid adding a separate deletion job, replay table, or outbox unless the contract requires it.

## Expired Uploads and Exports

- Select a bounded set of expired tracking rows and use their persisted object keys. Never reconstruct old keys from a newer naming
  convention.
- Recheck pending-upload eligibility before S3 work so a claimed file is not deleted. Preserve the repository's atomic claim/expiry
  integrity mechanism; do not hold a transaction open during storage calls.
- Delete S3 objects first, then remove only the tracking rows whose objects were successfully deleted. Keyless failed jobs may be removed
  without storage work.
- Retain failed candidates for another run. Rotate attempts when needed so persistent failures do not starve later rows.
- S3 lifecycle rules may provide a secondary safeguard, but do not report application object deletion merely because a database job expired.

## Storage Outcomes and Permissions

### Expired Export Cleanup Example

These dependency interfaces are illustrative application seams, not Storm APIs. The storage implementation returns IDs only for successful
per-object deletions (including verified absence), checks S3 batch errors, and throws on transport failure. The repository selects expired
jobs and rechecks expiry when deleting rows. Adapt those methods to existing services rather than introducing duplicate wrappers.

```ts
interface ExpiredExport {
	id: string;
	objectKey: string | null;
}

interface CleanupObject {
	id: string;
	objectKey: string;
}

interface ExportCleanupRepository {
	findExpired(limit: number): Promise<ExpiredExport[]>;
	deleteExpired(ids: string[]): Promise<number>;
}

interface CleanupStorage {
	deleteObjects(objects: CleanupObject[]): Promise<string[]>;
}

interface CleanupLogger {
	error(message: string): void;
}

export const cleanupExpiredExports = async (
	repository: ExportCleanupRepository,
	storage: CleanupStorage,
	logger: CleanupLogger,
	batchSize: number,
): Promise<number> => {
	const jobs = await repository.findExpired(batchSize);
	const keylessIds: string[] = [];
	const objects: CleanupObject[] = [];

	for (const job of jobs) {
		if (job.objectKey === null) {
			keylessIds.push(job.id);
		} else {
			objects.push({ id: job.id, objectKey: job.objectKey });
		}
	}

	let deletedIds: string[] = [];

	if (objects.length > 0) {
		try {
			deletedIds = await storage.deleteObjects(objects);
		} catch {
			logger.error('Export object cleanup failed; jobs retained for retry.');
		}
	}

	const deleted = new Set(deletedIds);
	const successfulIds = objects.filter((object) => deleted.has(object.id)).map((object) => object.id);

	// Keep failed objects reachable through their tracking rows for the next cleanup run.
	const removed = await repository.deleteExpired([...keylessIds, ...successfulIds]);

	return removed;
};
```

The returned count means tracking rows removed, not every candidate processed. Add per-object failure reporting using the existing logger.
If database deletion fails after S3 deletion, the retained row is safe to retry because already-missing objects count as complete. This
export example does not replace the eligibility/claim protections required for pending attachment uploads.

Handle individual failures in S3 batch responses even when the request itself succeeds. A verified already-missing object may count as
complete; permission and transport errors may not. Check bucket versioning semantics if physical version removal is required.

Grant cleanup access to every applicable prefix, including attachments and exports, without broadening unrelated permissions. Never send API
authorization tokens to presigned S3 URLs.

Report completed versus retained/failed work honestly. Caught failures need durable application retries because invocation-level retries
will not see them. Test successful, missing, partial-failure, and transport-failure cases, asserting both storage outcomes and retained
database state.
