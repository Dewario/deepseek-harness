# Independent audit report — github-agent-windows-subprocess-20261007

## Metadata
- Repository: `Dewario/deepseek-harness`
- Audit scope: subprocess spill handling, Windows process/shell paths, filesystem and directory-picker reliability
- Authoring mode: independent source-based audit (no production code changes)
- Provider/model/session identity: not authoritatively exposed by this runtime
- Audited head (expected): `2d2307e4d0f43bac355ff83814e9f921482401e5`
- Audited head (actual checkout): `2d2307e4d0f43bac355ff83814e9f921482401e5`
- PR/base lineage checked locally: `origin/master` at `c389f96bf3a9b6807cb71ed6bdad5849be0df6d8`
- Pinned upstream comparison baseline: `deepseek-ai/deepseek-harness@5badb15009ae1756c3afe0ae0cef1faafc290ccc`

## Verified change scope
`corepack pnpm --silent run change-scope --base origin/master --head 2d2307e4d0f43bac355ff83814e9f921482401e5`

Reported committed paths:
- `packages/subprocess/subprocess-local/src/spawn.ts`
- `packages/subprocess/subprocess-local/tests/spawn.spec.ts`

## Independent findings

### F1 — Recreated spill directory loses private-directory guarantees
- Severity: **Medium (P2)**
- Confidence: **High**
- Location:
  - `packages/subprocess/subprocess-local/src/spawn.ts:222-224` (`mkdirSync(this.spillDir, { recursive: true })`)
  - Contract context: `packages/subprocess/subprocess-local/src/spawn.ts:95-106`
  - Defensive pattern context: `docs/defensive-patterns.md:27-30`
- Impact:
  - On ENOENT during spill open/write, the fallback recreates the spill directory using default mode (umask-derived), not explicit private permissions.
  - This violates the "private spill directory" guarantee and can weaken ownership constraints of the spill location.
- Reproduction evidence (actual run in this session):
  - Command:
    - `node --import tsx/esm -e "... OutputCollector ... process.umask(0o022) ... process.umask(0o000) ..."`
  - Output:
    - `umask022 mode=755`
    - `umask000 mode=777`
- Minimal correction:
  - Do not recreate a cached deleted spill directory path in place.
  - Align with upstream spill ownership policy (see disposition table) where spill failure degrades to in-memory tail and reports failure.
- Owning tests/consumers:
  - Changed tests in this branch cover degradation/recovery shape but not permission guarantees:
    - `packages/subprocess/subprocess-local/tests/spawn.spec.ts:616-655`
  - Consumer chain:
    - `packages/subprocess/subprocess-local/src/spawn.ts:505-513`
    - `packages/subprocess/subprocess-local/src/index.ts:164-187`
    - `packages/shell/pwsh-local/src/index.ts:230-243,265-279,286-356`

### F2 — Spill ownership publication order can unlink a non-owned preexisting entry after failed exclusive open
- Severity: **Medium (P2)**
- Confidence: **Medium-High** (source-logic validated; no exploit claim)
- Location:
  - Candidate path published before exclusive open:
    - `packages/subprocess/subprocess-local/src/spawn.ts:204-209`
  - Shared error handler cleanup path:
    - `packages/subprocess/subprocess-local/src/spawn.ts:212-220,253-256`
- Impact:
  - `this.spillFile` is assigned before `openSync(...,'wx')` succeeds.
  - If open fails for a preexisting conflicting entry (`EEXIST` path), cleanup path can unlink the published path although ownership was not acquired.
- Minimal correction:
  - Publish `spillFile` only after successful exclusive open and ownership acquisition.
  - Keep cleanup limited to collector-owned file descriptors/paths.
- Owning tests/consumers:
  - This branch does not add a targeted ownership-collision test.
  - Existing related test only checks non-ENOENT fallback shape (`spawn.spec.ts:643-655`) and does not assert foreign entry preservation.

## Test/evidence verification notes

### Recovered focused test run (actual)
Command:
- `corepack pnpm exec vitest run packages/subprocess/subprocess-local/tests/spawn.spec.ts -t "degrades to the tail and recreates the dir when a cleaner deleted the spill dir|recovers spilling after the guard recreated a deleted spill dir|contains non-ENOENT spill failures and warns once across collectors"`

Result:
- `Test Files 1 passed`
- `Tests 3 passed | 88 skipped`

### Important correction about test-title interpretation
- `packages/subprocess/subprocess-local/tests/spawn.spec.ts:643` names the case "warns once across collectors".
- The body does **not** spy on or count `console.warn` calls.
- Therefore this test does **not** prove warn-once behavior.
- It also does not prove live-child stream survival semantics or native Windows behavior.

## Windows shell/process + directory-picker reliability trace

Reviewed source paths:
- `packages/subprocess/subprocess-local/src/windows-job.ts`
- `packages/subprocess/subprocess-local/src/index.ts`
- `packages/shell/pwsh-local/src/index.ts`
- `packages/host/directory-picker-browse/src/index.ts`
- `packages/host/directory-picker-native/src/win32-dialog-bindings.ts`
- `packages/api/workspace-controller/src/directory-picker.ts`

Observed status in this pass:
- No new concrete defect established in these reviewed paths beyond spill findings above.
- Ownership/cancellation defenses are present in examined paths (e.g., cancellable directory listing and Windows runner containment paths).
- This audit did not execute native Windows or dialog-worker runtime tests.

## TODO / placeholder classification
Searched tokens in reviewed files: `TODO`, `FIXME`, `XXX`, `placeholder`, `not implemented`.

Result:
- No unfinished-required-behavior marker was found in audited source for spill/Windows picker logic.
- One match was a test command fragment (`wait_placeholder=`) at:
  - `packages/subprocess/subprocess-local/tests/spawn.spec.ts:824`
- Classification: benign shell variable name in test fixture, **not** unfinished behavior.

## Upstream comparison disposition (vs pinned upstream `5badb150...`)

| Item | Local branch status (`2d2307e...`) | Upstream status (`5badb150...`) | Disposition |
|---|---|---|---|
| Spill-dir recreation on ENOENT (F1) | Present in `spawn.ts` | Upstream spill logic moved to `packages/subprocess/subprocess-local/src/output.ts` and documents no recreation of deleted cached dir | **Already fixed upstream / not to carry forward** |
| Spill path publication before open (F2) | Present in `spawn.ts` | Upstream opens into local variable first, then publishes owned path/fd in `output.ts` | **Already fixed upstream / not to carry forward** |
| Branch non-ENOENT test title implies warn-once | Title present; body lacks warning assertions | Upstream tests are reorganized around `output.ts` and include explicit spill failure reporting coverage | **Retained in branch test wording; superseded by upstream test structure** |

## Codex PR #1 claim verification in this scope

Relevant tracking comment source reviewed:
- PR #1 issue comment `6043406792` (Codex windows/subprocess report)

Verification result in this independent pass:
- Spill directory permission regression claim: **verified**.
- Spill ownership-publication-order risk claim: **verified by source logic**.
- Any claim of native-Windows runtime proof from these branch tests: **not verified; not supported by executed evidence here**.

## Implementation-ready acceptance criteria (report only; no code changes performed)

1. **Spill failure policy**
   - A spill open/write failure must never crash stream data handling.
   - In-memory tail collection must continue and `finalize()` remain usable.

2. **Directory ownership/security semantics**
   - Deleted cached spill directory path is not recreated with ambient/default permissions.
   - If a recovery strategy is retained, it must preserve private ownership guarantees explicitly and be validated.

3. **Spill file ownership lifecycle**
   - Collector only unlinks files it successfully created/opened.
   - Exclusive-open collision path (`EEXIST`) must preserve preexisting foreign entry.

4. **Test clarity and evidence alignment**
   - Test names must match assertions; warn-once behavior requires explicit warning-count assertions.
   - Native-Windows claims require native-Windows execution evidence.

5. **Upstream integration safety**
   - When integrating from this stale branch, prefer upstream `output.ts` behavior at pinned baseline and avoid reintroducing branch-only regression paths.

## Commands executed (this session, relevant subset)

1. Verify checkout and refs:
- `git rev-parse HEAD`
- `git branch --show-current`
- `git fetch origin master:refs/remotes/origin/master --quiet`
- `git rev-parse origin/master`

2. Scope check:
- `corepack pnpm --silent run change-scope --base origin/master --head 2d2307e4d0f43bac355ff83814e9f921482401e5`

3. Focused tests:
- `corepack pnpm exec vitest run packages/subprocess/subprocess-local/tests/spawn.spec.ts -t "degrades to the tail and recreates the dir when a cleaner deleted the spill dir|recovers spilling after the guard recreated a deleted spill dir|contains non-ENOENT spill failures and warns once across collectors"`

4. Mode probe:
- `node --import tsx/esm -e "...OutputCollector umask probe..."`

## Reviewed vs unreviewed scope

Reviewed:
- Full changed files in branch diff (`spawn.ts`, `spawn.spec.ts`) and relevant consumers in subprocess runtime + pwsh executor + directory-picker controller/native/browse paths listed above.

Not reviewed or not executed:
- Native Windows runtime execution paths (real Win32 process/job and dialog runs).
- Full repository suite, full CI matrix, and exhaustive cross-package regression testing.
- Production credentials/user data (none accessed).

## Constraints honored
- No production code changes.
- No merge/rebase/history rewrite/default-branch or settings changes.
- No full-suite run by default.
- No credential access/output.
