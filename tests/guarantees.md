# Concurrency and Failure Guarantees

Sources: [Commits](../docs/commits.md), [Database Configuration](../docs/database.md),
[Pages and Cursors](../docs/pagination.md), and [Testing Plan](README.md).

These are requirements that the ordinary transcripts do not fully establish.
They need environment-specific observation or controlled schedules. They remain
part of the specification; no empty test or skipped assertion counts as coverage.

## Atomic Conditional Writes

A conditional set or delete compares the stored ETag and updates the record and
indexes atomically. Competing callers cannot both successfully replace the same
version without another read. A failed condition writes nothing.

[Retries](retries.md) and [Mutations](mutations.md) produce deterministic stale
versions through another real connection. [Ownership](relationships.md) runs two
concurrent creation requests. These observations do not exhaust all schedules.

## Gate a Whole Batch

Capture final object values at exit and finish local validation/serialization
before writes. Check every condition against initial storage state immediately
before execution. No client can intervene between checks and writes. Failed
conditions cancel every write, even unconditional ones. Operations then run in
order; conditions are not rechecked. Each record operation maintains its indexes.

The boundary is one Redis primary or Cluster hash slot. Cross-shard transactions,
nested commits, and savepoints are unsupported. Arbitrary execution errors have
no general rollback guarantee. Persistence and failover durability depend on
deployment. See Redis [transactions](https://redis.io/docs/latest/develop/using-commands/transactions/),
[scripting](https://redis.io/docs/latest/develop/programmability/eval-intro/), and
[Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/).

## Lose Confirmation

Use a controlled real transport to interrupt delivery around a write's response.
Observe StorageError and unresolved Result handles. Verify no automatic replay,
then inspect storage independently. It may already contain the write.

Do not infer absence of writes from the exception. Local ETags and database
associations remain unchanged after an unknown outcome. Result access raises
CommitError without waiting or executing. Original failures propagate.
This transport scenario has no implemented harness yet.

## Race Primary Creation

[Store and Retrieve](storage.md#create-from-competing-requests) starts competing
requests on independent connections. It asserts one winner and consistent
returned values without selecting which request wins. A barrier at the client
does not force every relevant storage interleaving.

Use controlled schedules for stronger concurrency checks. Get-or-create must produce one stored record: the winner returns created=True,
others return that record with created=False. Existing fields and ETag remain
unchanged. Creation defaults cannot overwrite identity. The email workflow's
IF_MISSING conflict behavior differs from get-or-create and add.

## Observe Connection Ownership

Observe a supplied real client's pool and transport while leaving execution
intact. Check laziness before first storage access, owned-client close/reopen,
borrowed-client preservation, binary responses, disabled command retries, TLS
verification, timeout behavior, and no fallback routing on failure.

Thread sharing uses pooling; instances require application synchronization.
Processes must create their own live clients. Defaults/registrations are
process-local. Default changes and rebinding occur at startup, not concurrently.
The API is synchronous. There is no cancellation/draining on close.

## Observe Storage Work

Constructor/decoder counts establish model loading, not absence of server scans.
Observe actual commands, requested ranges, and script execution where needed.
Unsupported queries must fail rather than scan or filter loaded records. Pages
continue by index position rather than offsets and may inspect one extra index
entry without fetching its record. First/last must not load intervening records.

Use increasingly large datasets as supporting evidence. Do not assert exact
command sequences or treat latency/transfer size alone as proof. Document any
visibility gap. Index decoding is separate from record decoding.

## Observe Iterator Lifetime

The iterator buffers at most one page, releases references to previous pages,
and makes no background requests. No new request follows stopped consumption.
No count or full-result materialization is allowed. A concurrent delete may
produce an empty page with a cursor; continuation must advance and follow that
cursor rather than stop early. Ordering/filter changes can skip or repeat records.

## Observe Failures Without Mocks

Use real constructors, validators, registered encoders/decoders, copy protocols,
and application callbacks that intentionally fail. Delegate normal work when
measuring success. Confirm observers detect known work before asserting absence.
Compare instrumented and uninstrumented results, excluding setup work.

Page failure yields no partial page. Iterator failure preserves earlier yielded
pages but yields nothing from the failing page and terminates without retry.
Refresh failure preserves the original. Mutation copying fails before callbacks
or writes; confirmed transfer repeats neither callbacks nor construction.

## Keep Coverage Honest

The Markdown examples have not run against an implementation. Static parsing and
link checks cannot establish these guarantees. Backend failure tests, full
concurrency scheduling, cross-process runs, engine/version matrices, and reliable
cleanup on test failure remain future harness work, not new public API.
