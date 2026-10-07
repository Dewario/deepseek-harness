# github-agent-core-persistence-20261007

## Scope and provenance

This report is a read-only independent audit of the core execution and JSONL persistence paths requested for tracking PR #1. It records source inspection and focused tests; it does not certify the repository, implement fixes, or approve integration.

## Verified refs

- Audited source baseline: `Dewario/deepseek-harness@2d2307e4d0f43bac355ff83814e9f921482401e5`.
- Tracking PR #1 at review start: base `c389f96bf3a9b6807cb71ed6bdad5849be0df6d8`; head `2d2307e4d0f43bac355ff83814e9f921482401e5`.
- Current report branch at audit start: `copilot/spill-dir-deletion-guard-again`, with report-only commit `3dd34a87467b74e1241d20197f8f6bc2fd4c9881` on top of the exact audited baseline.
- Pinned upstream comparison: `deepseek-ai/deepseek-harness@5badb15009ae1756c3afe0ae0cef1faafc290ccc`.

The PR #1 scope command was run against its verified base and target head. It found only `packages/subprocess/subprocess-local/src/spawn.ts` and `packages/subprocess/subprocess-local/tests/spawn.spec.ts`; this audit remains independent of that diff.

## Commands run and results

1. `git show --no-patch --format='%H %P' 2d2307e4d0f43bac355ff83814e9f921482401e5`
   - Result: target commit `2d2307e4d0f43bac355ff83814e9f921482401e5`, parent `c389f96bf3a9b6807cb71ed6bdad5849be0df6d8`.
2. `corepack pnpm --silent run change-scope --base c389f96bf3a9b6807cb71ed6bdad5849be0df6d8 --head 2d2307e4d0f43bac355ff83814e9f921482401e5`
   - First attempt failed because the shallow checkout had no merge base. After `git fetch --unshallow --no-tags origin`, the identical command passed and reported the two subprocess paths listed above.
3. `corepack pnpm --silent exec vitest packages/core/agent-loop/tests/tool-calls.spec.ts -t 'stops new dispatches and drains started bodies before surfacing the first failure' --run`
   - Passed: 1 test; 20 skipped.
4. `corepack pnpm --silent exec vitest packages/core/session/tests/repair.spec.ts -t 'synthesized tool/result carries surfaceOp and sourceEventSeqs when tool/call was logged' --run`
   - Passed: 1 test; 10 skipped.
5. `corepack pnpm --silent exec vitest packages/core/agent-loop/tests/resume.spec.ts -t 'resume closes an interrupted tool call durably' --run`
   - Passed: 1 test; 44 skipped.
6. `corepack pnpm --silent exec vitest packages/core/agent-loop/tests/loop.spec.ts -t 'abandons the live row when durable Assistant settlement is rejected|abandons a started live attempt when final block assembly fails|preserves the stream failure when its durable attempt settlement is also rejected' --run`
   - Passed: 3 tests; 61 skipped.
7. `corepack pnpm --silent exec vitest packages/core/agent-loop/tests/cancel.spec.ts -t 'stream' --run`
   - Passed: 7 tests; 32 skipped.
8. `corepack pnpm --silent exec vitest packages/session/session-persistence-jsonl/tests/generation.spec.ts -t 'retains a POSIX publication after the directory sync fails' --run`
   - Passed: 1 test; 69 skipped. This is the historical-generation migration publisher, not first materialization through `JsonlSessionHandle`.
9. `corepack pnpm --silent exec vitest packages/session/session-persistence-jsonl/tests/jsonl.spec.ts -t 'a failed appendLines truncates partial bytes so a retry has no seq gap' --run`
   - Passed: 1 test; 170 skipped.
10. `corepack pnpm --silent exec vitest packages/session/session-persistence-jsonl/tests/lease.spec.ts -t 'a drain failure and a release failure reject close as one AggregateError' --run`
    - Passed: 1 test; 18 skipped. The test injects a backend persistence rejection on a real handle; it checks failure aggregation and ownership release, not whether the buffered event survives.
11. `corepack pnpm --silent exec vitest packages/session/session-persistence-jsonl/tests/jsonl.spec.ts -t 'publishes v2 beside an unchanged physical v1 source|selects v1 from a v0/v1 directory, then v2 from the retained three-generation set|opens the migrated successor for append while retaining the historical source' --run`
    - Passed: 3 tests; 168 skipped.
12. `corepack pnpm --silent exec vitest packages/session/session-persistence-jsonl/tests/generation.spec.ts -t 'publishes only the final generation across a multi-edge migration|uses one immutable publication algorithm for' --run`
    - Passed: 3 tests; 67 skipped.

Vitest printed the repository's existing `vite-tsconfig-paths` deprecation warning. No full suite, coverage, build, lint, Windows-native run, credential-backed test, or real-provider request was run.

## Findings

### GACP-CORE-01 — P1, high confidence: terminal scheduler failure closes a step with unanswered assistant tool calls

**Locations at the audited baseline**

- `packages/core/agent-loop/src/agent.ts:457-475` commits the assistant message before invoking `executeToolCalls`; `agent.ts:290-304` closes the step in `finally`, and `agent.ts:327-334` closes the turn.
- `packages/core/agent-loop/src/tool-calls.ts:219-236` stops scheduling, drains in-flight work, then rethrows a terminal scheduler failure without recording results for unanswered calls. Ordinary tool failures are normalized by the execution path; this finding concerns the exceptional scheduler failure.
- `packages/core/session/src/repair.ts:50-53,79-82` clears pending calls at `step/end` and returns no closers for an already-balanced turn.
- Existing failure-quiescence test: `packages/core/agent-loop/tests/tool-calls.spec.ts:639-696`.

**Impact and evidence level**

The committed assistant message can contain tool requests without a matching `tool/result`, followed by `step/end` and `turn/end`. Repair correctly treats the log as balanced and does not recover requests from the closed step. A later request can therefore project an invalid provider transcript. The existing failure test proves dispatch quiescence and error-turn settlement; it does not assert one result per requested call or validate a subsequent provider request. The targeted repair and resume tests pass for interrupted open tails, not this already-closed-step case.

**Minimal design and acceptance**

Integrate the pinned upstream recovery mechanism rather than synthesize “not dispatched” results for every call. Upstream `packages/core/agent-loop/src/agent.ts:331-356` observes committed events with `ToolCallRecovery` and appends its missing results before `step/end`; `packages/core/session/src/repair.ts` distinguishes `TOOL_NOT_STARTED` from `TOOL_OUTCOME_UNKNOWN` and retains `sourceEventSeqs` for started calls. Preserve the first scheduler error if settlement also fails.

Extend the failure-quiescence test to assert exactly one matching result for every assistant tool-call before step closure, correct not-started/unknown classification, completed child settlement, and correct source references. Add a resumed/follow-up request test through the composed Harness and verify the provider accepts the reconstructed transcript. Update model-visible snapshots and both SDK expected outputs if the integrated changes alter their projected output.

**Upstream disposition:** Fixed at `5badb15009ae1756c3afe0ae0cef1faafc290ccc` by `ToolCallRecovery` in the AgentLoop step owner and expanded failure tests. The naive “append skipped results after drain” suggestion in the independent agent report is insufficient: a started call can have an unknown side-effect outcome.

### Stream settlement: no additional defect established

`packages/core/agent-loop/src/agent.ts:382-422,449-479` settles an interrupted assistant message or failed attempt before leaving the step. `packages/core/agent-loop/src/assistant-stream.ts:73-108` emits terminal `end` only after the durable event append succeeds, otherwise emits `abandoned`; combined stream and settlement failures retain both errors. The executed loop and cancellation tests cover settlement rejection, assembly failure, cancellation, and error preservation. No separate stream-settlement blocker was established. Cancellation plus a rejected interrupted-message append remains a narrower untested combination.

### GACP-PERSIST-02 — P2, high confidence in control flow; full-handle fault injection remains unverified: first-materialization directory-sync failure wedges same-handle retry

**Locations at the audited baseline**

- `packages/session/session-persistence-jsonl/src/index.ts:807-821` calls `materialize()` for an unmaterialized writer and advances tracker state only after it resolves.
- `index.ts:1135-1148` links the temp file to `finalPath` before syncing the session directory. A sync rejection leaves the full candidate path present but rejects the operation.
- `packages/session/session-persistence-jsonl/src/storage.ts:318-343` advances `materialized` and `cursor` only after persistence resolves; `storage.ts:294-315` restores a failed live batch to its buffer.
- `index.ts:1182-1191` refuses the next materialization because the log now exists. A same-handle append, flush, or close drain therefore cannot finish this transaction. A fresh open can read the existing candidate, but it does not prove that the earlier directory entry became crash-durable.

**Correction to Codex evidence**

This is a recoverability/durability-state gap, not demonstrated lost or corrupted bytes. The file may be present and readable; namespace durability remains unconfirmed after the failed `fsync`. PR #1's GPT-6 Astra verification reports an extracted `materializePosix` probe using a real temporary directory and a post-link sync fault; retry hit the existing-log refusal while the artifact and modeled cursor/buffer remained. That probe is not the actual `StorageHandle`/consumer composition. The local test run above exercises `generation.spec.ts:1165-1188`, which tests a separate historical-generation migration publisher and fresh migration retry, not first materialization through `JsonlSessionHandle`. Do not cite that passing test as proof of same-handle recovery or failure.

The pinned upstream `packages/session/session-persistence-jsonl/src/index.ts` retains the same POSIX `link()` then directory-sync order (`1215-1226`) and the storage state still advances only after `persistBatch` resolves. This issue remains upstream.

**Minimal design and acceptance**

Prefer one pending-publication transaction owned by the same write handle and lease. Retain the original encoded candidate and publication identity after link; on retry verify the owned regular file and exact original bytes, retry only the missing directory sync, and advance cursor/materialized state once after that barrier succeeds. Refuse substituted, modified, truncated, extended, or unrelated artifacts without overwrite or deletion. Do not set `materialized` in a catch. A fail-closed/reopen policy is an alternative only if it defines how the unsynced namespace and retained events are handled.

Add an owning POSIX test using the actual JSONL provider and `JsonlSessionHandle`, with a path/phase-specific fault after successful `link()` and real directory sync. Assert first failure, unchanged cursor, present exact file, same-handle retry/flush behavior, duplicate-free readback and next-sequence append, repeated failures, an altered/replaced candidate refusal, concurrent live events, close/lease behavior, header-only flush, and unchanged predecessor-generation bytes. Cover both raw and Zstandard codecs where the shared implementation applies. The migration-only test is a control, not acceptance for this transaction.

### GACP-PERSIST-03 — P2, high confidence in provider control flow; consumer-loss evidence is incomplete: failed close strands routed live events

**Locations at the audited baseline**

- `packages/session/session-persistence-jsonl/src/storage.ts:294-315` restores a failed live batch to the handle buffer.
- `storage.ts:223-259` caches the close promise, records a drain failure, then releases the lease and in-process handle claim regardless of that failure.
- `storage.ts:495-500` removes the handle and writer mapping; it also deletes a never-materialized pending session.
- `storage.ts:535-552` routes `session/event` into the buffer and starts `close()` on `session/disposed`, logging a drain failure rather than keeping a retry route.
- The close JSDoc at `storage.ts:214-220` promises a drain that loses nothing through teardown.
- `packages/session/session-persistence-jsonl/tests/lease.spec.ts:391-406` injects a drain refusal on a real provider handle, then proves close reports errors and a new writer can acquire the id. It does not reopen and assert what happened to the buffered event, and it manually calls `enqueueLive` rather than exercising `Session` disposal.

After a close drain rejects, the cached close promise cannot be retried and the provider no longer routes new events to that handle. Its buffered batch is not durable and is no longer reachable through the active writer map. The direct close caller receives a rejection; `session/disposed` only logs it. Do not describe this as silent successful persistence or as data already present on disk. If the process exits or the session is disposed, the buffered event has no demonstrated recovery path, contrary to the local lossless-teardown comment.

**Minimal design and acceptance**

Define one policy for failed close: either keep the failed writer/buffer reachable for a retry while preserving ownership, or propagate an explicit undurable-event failure to a lifecycle owner that can retain/replay the events. Reconcile the close JSDoc and `session/disposed` handling with that policy; do not simply release the only in-memory copy and call close lossless. Add an owning composition test that publishes a real session event, faults its persistence write, invokes session disposal and backend teardown, then proves either successful later durable replay or an explicit, externally observable undurable disposition. Also retain lease-release and aggregate-error assertions.

**Upstream disposition:** The pinned upstream storage implementation has the same close-drain/release pattern. No fix was found.

### GACP-PERSIST-04 — P2, high confidence in control flow; retry reproduction absent: failed append rollback leaves the writer cursor reusable

**Locations at the audited baseline**

- `packages/session/session-persistence-jsonl/src/index.ts:1246-1270` appends bytes, then on write/fsync failure closes the file and attempts `rollbackAppend`. If rollback itself fails, it throws an `AggregateError`.
- `packages/session/session-persistence-jsonl/src/storage.ts:318-343` leaves the cursor unchanged when `persistBatch` rejects, and `storage.ts:323` validates the next batch against that unchanged cursor.
- Existing tests: `packages/session/session-persistence-jsonl/tests/jsonl.spec.ts:1568-1604` proves retry after a successful rollback; `jsonl.spec.ts:1606-1642` proves both write and rollback failures are reported, but does not retry the same handle afterward.

When rollback fails, the handle does not mark its physical tail uncertain or prevent another append. A retry with the same sequence numbers passes the unchanged in-memory cursor check and writes at end-of-file, potentially duplicating a complete batch or following a partial tail. This is not evidence of corruption from every fsync failure: the tested successful-rollback path remains retryable. It is a distinct failure-after-failure path.

**Minimal design and acceptance**

After rollback failure, either repair/rescan the physical tail against the committed cursor before accepting another append, or fail the handle closed and require an explicit reopen. Preserve the original append and rollback errors. Fault-inject append fsync failure followed by rollback failure, then retry the same batch and a later batch; assert no duplicate append, no invalid log acceptance, and a valid read/reopen result under the selected policy.

**Upstream disposition:** The pinned upstream `appendLines()` still attempts rollback and throws on rollback failure (`index.ts:1324-1352`); the handle has no corresponding uncertain-tail state. Not fixed upstream.

## Verified correct behavior and intentional limitations

### Stream and transcript recovery

Stream attempts settle or explicitly abandon their live presentation before the step exits. Crash repair correctly emits ordered `TOOL_NOT_STARTED` or `TOOL_OUTCOME_UNKNOWN` results for unmatched calls in an open tail, includes `surfaceOp` and source sequences when a `tool/call` exists, and closes step before turn. It intentionally returns no repair for a balanced log; CORE-01 is the missing live scheduler-failure recovery before that boundary, not a defect in the balanced-log rule.

### Writer ownership

One write handle per session is an intentional contract. `packages/session/session-persistence-jsonl/src/lease.ts:1-27,68-145` uses an in-process claim plus POSIX `flock` or a Windows named semaphore, releases kernel ownership on process death, and deliberately has no expiry that could let a second writer overlap a live stalled writer. POSIX removal of `session.lock` forfeits the inode-based exclusion; README.md:156 documents this. Release failure can still reject close while freeing the in-process claim (`storage.ts:243-258`); tests cover aggregate reporting and retrying ownership.

Lazy create, explicit empty-session flush, routed batching, and same-handle retry after a successful append rollback are intentional behaviors covered at `tests/jsonl.spec.ts:1223-1252,1568-1604`. A possible live-buffer memory bound during a prolonged storage outage is an operational policy question, not a defect established by this audit.

### Migration and released-generation immutability

The adjacent migration path stages and verifies a new current file, checks source identity before publication, and publishes without overwrite (`packages/session/session-persistence-jsonl/src/generation.ts:879-938`). Tests prove final-only publication, unchanged predecessor bytes/identity, and selection of the highest retained generation (`tests/generation.spec.ts:823-863`; `tests/jsonl.spec.ts:886-930`). Read-only migration preparation does not publish a successor (`jsonl.spec.ts:683-705`). These behaviors comply with the released-format rule: committed generations are not moved, overwritten, or deleted; retained predecessors imply no fallback or downgrade support. No migration or released-generation defect was found in this scope.

Documented constraints include single live writer, lazy materialization, supported adjacent migration edges, immutable predecessors, no automatic fallback/downgrade, and best-effort removal of the redundant temporary hard link only after publication is durable (`packages/session/session-persistence-jsonl/README.md:72-78,145-157`; `generation.ts:800-809`). They are not findings.

## Codex PR #1 report/spec corrections

- **CORE-01:** Confirmed at the pinned fork baseline; exceptional scheduler failure only, not ordinary tool-body errors or the normal cancellation path. Upstream has the complete `ToolCallRecovery` behavior. Do not use a simple synthetic “skipped” result for started calls because their side-effect outcome can be unknown.
- **CORE-02:** Confirmed as a same-handle retry and durability-state gap. Preserve the correction that it does not demonstrate event loss or corruption. The prior exact-method probe is component-level; the local passing `generation.spec.ts` sync-failure test is a separate migration path. The Codex spec correctly requires actual handle composition, original-candidate identity, exact-once cursor advancement, and predecessor immutability.
- **Spec gap:** The proposed close/teardown acceptance must resolve GACP-PERSIST-03: persistent drain failure currently releases the only active writer while its buffer remains inaccessible. Add the real session-event/disposal regression before calling that behavior lossless.
- **Additional path:** GACP-PERSIST-04 extends rollback coverage beyond the current successful-rollback retry test. A rollback-failure `AggregateError` does not currently poison or repair the same handle before a subsequent append.

## Disjoint implementation ownership

1. **Core transcript settlement owner:** `packages/core/agent-loop/src/agent.ts`, `packages/core/agent-loop/src/tool-calls.ts`, `packages/core/session/src/repair.ts`, their focused tests, and any required session/agent README pair. Integrate the upstream helper coherently. Own required model-visible snapshots and TypeScript/Python SDK expected outputs if the resulting projection changes.
2. **JSONL failure-state owner:** `packages/session/session-persistence-jsonl/src/index.ts`, `src/storage.ts`, `tests/jsonl.spec.ts`, and `tests/lease.spec.ts`. This one owner handles GACP-PERSIST-02/03/04 because their commit, buffer, and close state share these files. Update `README.md`, `README.zh.md`, and `README.i18n.yaml` if caller-visible retry/close semantics change. Keep `src/generation.ts` and `tests/generation.spec.ts` out unless the chosen implementation actually changes migration publication.
3. **Integration owner:** owns only cross-package composed fixtures, shared SDK/snapshot outputs, and final dependency coordination when needed. Do not split the shared JSONL handle state across competing persistence workstreams. This report author owns no production files.

## Remaining coverage gaps and disposition

- No real StorageHandle test injects the post-link first-materialization directory-sync failure; the previous standalone probe does not cover that composition.
- No same-handle retry follows an append fsync failure plus rollback failure.
- Existing drain-close failure coverage does not assert buffered event fate and does not invoke actual Session disposal.
- No real-provider request validates a follow-up transcript after CORE-01. Focused unit/resume tests do not substitute for that acceptance case.
- No native Windows matrix, full repository suite, coverage gate, built-profile smoke, real-provider run, or full upstream repository diff was executed. Upstream conclusions are targeted source comparisons at the pinned SHA, not a whole-tree merge-readiness assessment.

These gaps leave four findings for implementation-owner triage: CORE-01 is a P1 transcript-integrity defect already fixed upstream; PERSIST-02 is an upstream-unfixed P2 retry/durability gap; PERSIST-03 is an upstream-unfixed P2 failed-close retention gap; and PERSIST-04 is an upstream-unfixed P2 rollback-failure retry gap. No production fix, merge, default-branch/settings change, history rewrite, or user-data mutation was performed.
