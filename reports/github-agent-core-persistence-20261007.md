# github-agent-core-persistence-20261007

## Provenance
- Report authoring provider: GitHub Copilot Task Agent
- Report authoring model/session: runtime-managed model (name not exposed in tool runtime), session at 2026-10-07T22:28Z
- Repository: `Dewario/deepseek-harness`
- Audit mode: independent source + targeted existing-test evidence (no production-code edits)

## Verified refs
- Local checkout HEAD (verified): `2d2307e4d0f43bac355ff83814e9f921482401e5`
- Tracking PR #1 base/head: base `c389f96bf3a9b6807cb71ed6bdad5849be0df6d8`, head `2d2307e4d0f43bac355ff83814e9f921482401e5`
- Upstream comparison pin: `deepseek-ai/deepseek-harness@5badb15009ae1756c3afe0ae0cef1faafc290ccc`

## Scope completed
- Core agent-loop exceptional scheduling and stream/step settlement behavior
- Durable transcript balancing and resume repair behavior
- JSONL persistence append/publication/fsync failure behavior
- Ownership/lease/disposal + retry/flush/close behavior
- Migration and released-generation immutability handling
- Verification/correction of relevant Codex PR #1 core/persistence claims

## Commands executed and outcomes
1. `git rev-parse HEAD && git branch --show-current && git status --short`
   - Result: HEAD `2d2307e...`, branch `copilot/spill-dir-deletion-guard-again`, clean worktree.
2. `corepack pnpm --silent run change-scope --base c389f96... --head 2d2307e...`
   - Result: changed paths in PR #1 are subprocess-only (`packages/subprocess/subprocess-local/...`), confirming this audit is independent of the narrow diff.
3. `corepack pnpm --silent exec vitest packages/core/agent-loop/tests/tool-calls.spec.ts -t "failure quiescence" --run`
   - Result: passed (1 test run, others skipped).
4. `corepack pnpm --silent exec vitest packages/core/session/tests/repair.spec.ts -t "synthesized tool/result carries surfaceOp and sourceEventSeqs when tool/call was logged" --run`
   - Result: passed.
5. `corepack pnpm --silent exec vitest packages/core/agent-loop/tests/resume.spec.ts -t "resume closes an interrupted tool call durably" --run`
   - Result: passed.
6. `corepack pnpm --silent exec vitest packages/session/session-persistence-jsonl/tests/generation.spec.ts -t "retains a POSIX publication after the directory sync fails" --run`
   - Result: passed.

## Findings

### GACP-CORE-01 (High, high confidence)
**Title:** Terminal scheduler failure can commit `step/end`/`turn/end` while leaving assistant-requested tool calls without durable `tool/result` settlement.

**Evidence (target SHA):**
- `packages/core/agent-loop/src/tool-calls.ts:232-236` throws scheduler failure after draining in-flight dispatches, with no synthetic recovery results for unresolved calls.
- `packages/core/agent-loop/src/agent.ts:290-304` always appends `step/end` in `finally` after `this.step(...)`.
- `packages/core/agent-loop/src/agent.ts:327-334` always appends `turn/end` in outer `finally`.
- `packages/core/session/src/repair.ts:50-53` clears pending call tracking at `step/end`; once `step/end` is present, crash repair cannot synthesize missing tool settlements for that closed step.

**Impact:**
- A stored session can contain an assistant tool-call request without matching `tool/result` events despite a closed step/turn.
- Follow-up request projection can carry unresolved assistant tool calls into provider transcripts, risking provider transcript rejection and ambiguous operator recovery.

**Repro/acceptance target:**
- Extend failure-quiescence coverage (`packages/core/agent-loop/tests/tool-calls.spec.ts` region around existing failure scenario) to assert: every tool call in the assistant message has exactly one durable result before `step/end`, and follow-up/resume remains provider-valid.

**Minimal fix design (no code change performed here):**
- Integrate the upstream `ToolCallRecovery` approach from `deepseek-ai/deepseek-harness@5badb150...` where the step owner appends conservative results in catch-path before `step/end`.

**Upstream disposition:**
- **Fixed upstream** (confirmed by upstream `packages/core/agent-loop/src/agent.ts` using `ToolCallRecovery`, plus expanded upstream tool-call failure tests).

---

### GACP-PERSIST-02 (Medium, medium confidence)
**Title:** First-materialization post-publication directory-sync failure can strand the in-memory writer state until handle reopen; this is a recoverability/availability gap, not demonstrated data loss.

**Evidence (target SHA):**
- `packages/session/session-persistence-jsonl/src/index.ts:1137-1148` publishes with `link(tmp, finalPath)` then syncs directory.
- On a directory-sync rejection after successful link, `persistBatch` rejects before `tracker.materialized(header.id)` (`index.ts:818-820`) and before handle state advancement (`storage.ts:339-342`).
- Retry path still in `isMaterialized=false` branch re-enters materialization and hits `rejectExistingLog` (`index.ts:1182-1191`) because file now exists.

**Important correction:**
- This path does **not** by itself prove committed data loss; it proves a writer-state retry wedge. The published file can still exist and be readable, and behavior depends on consumer reopen/recovery handling.

**Repro/acceptance target:**
- Fault inject exactly one `syncDirPosix(dir)` failure after successful `link()` publication, then restore FS health.
- Assert: (a) committed file presence unchanged, (b) same-handle retry/flush behavior is explicit and deterministic, (c) reopen path behavior is defined and tested, (d) no overwrite/delete of committed generations.

**Minimal fix design options (choose one explicitly):**
1. **Retry-completes-durability path:** persist publication identity and on retry complete pending durability barrier when artifact identity matches expected bytes.
2. **Fail-closed explicit-reopen contract:** mark handle terminal for writes after this specific failure mode, surface typed error, require reopen; document this as intentional and add contract tests.

**Upstream disposition:**
- **Not fixed upstream** at pinned comparison SHA; equivalent control flow remains.

## Intentional limitations vs unfinished promised behavior

### Intentional / documented limitations
- Single live writer per session (in-process claim + cross-process lease), including create-path lazy acquisition (`packages/session/session-persistence-jsonl/src/lease.ts`, README limitations).
- Released generation immutability and adjacent migration publication model are deliberate (`packages/session/session-persistence-jsonl/README.md`, architecture/Agent Notes).

### Unfinished/unclear promised behavior needing explicit product decision
- Post-publication directory-sync failure semantics for same-handle retries are not clearly codified as either retryable-complete or fail-closed-reopen; current behavior appears implicit via control flow rather than explicit contract.

## Codex PR #1 core/persistence claim verification
- `CORE-01` from Codex report: **confirmed**.
- `CORE-02` from Codex report: **partially confirmed and corrected** — keep as recoverability candidate; do **not** infer data loss solely from retry refusal in the same handle.

## Disjoint file ownership for fix-ready workstreams
1. **Workstream A (core scheduler settlement):**
   - `packages/core/agent-loop/src/agent.ts`
   - `packages/core/agent-loop/tests/tool-calls.spec.ts`
   - `packages/core/agent-loop/tests/resume.spec.ts` (acceptance coverage touchpoint)
2. **Workstream B (JSONL materialization failure semantics):**
   - `packages/session/session-persistence-jsonl/src/index.ts`
   - `packages/session/session-persistence-jsonl/src/storage.ts`
   - `packages/session/session-persistence-jsonl/tests/jsonl.spec.ts`
   - `packages/session/session-persistence-jsonl/tests/generation.spec.ts`
3. **Workstream C (contract/doc clarity if B chooses fail-closed):**
   - `packages/session/session-persistence-jsonl/README.md`
   - `docs/subsystems/persistence.md` (if service contract surface is changed/clarified)

## Coverage gaps in this audit
- No full-suite execution, no Windows native lane, no real-provider transcript validation run.
- No new fault-injection test was added in this report-only PR.
- Upstream pin was compared via targeted file retrieval at SHA; full repository diff across SHAs was not exhaustively executed.

## Final disposition
- One high-confidence actionable bug (GACP-CORE-01), fixed upstream and integration-ready.
- One medium-confidence persistence recoverability gap candidate (GACP-PERSIST-02) requiring explicit contract decision and targeted fault-injection acceptance before mandatory implementation.
