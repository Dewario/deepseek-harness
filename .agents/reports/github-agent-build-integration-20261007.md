# github-agent-build-integration-20261007

- **Repository:** `Dewario/deepseek-harness`
- **Audit scope date (UTC):** 2026-10-07
- **Audited local HEAD:** `2d2307e4d0f43bac355ff83814e9f921482401e5`
- **Tracking PR:** https://github.com/Dewario/deepseek-harness/pull/1
- **Pinned upstream comparison ref:** `deepseek-ai/deepseek-harness@5badb15009ae1756c3afe0ae0cef1faafc290ccc`
- **Provider/model/session identity (observed evidence):** GitHub Copilot cloud agent runs (`actor=Copilot`), model string `gpt-5.3-codex` present in failed run logs (`run_id=37695087569`, `job_id=113044722628`), with prior cloud task/session IDs listed in PR #1 comment `6048073623`.

## 1) Baseline and ref verification

### Verified refs
- Local checkout SHA equals expected SHA: `2d2307e4d0f43bac355ff83814e9f921482401e5`.
- Current local branch: `copilot/spill-dir-deletion-guard-one-more-time`.
- PR #1 head: `spill-dir-deletion-guard@2d2307e4...` and base: `master@c389f96bf3a9...`.
- Divergence counts:
  - `c389f96b...` vs `5badb150...`: **0 ahead / 20734 behind**.
  - `2d2307e4...` vs `5badb150...`: **1 ahead / 20734 behind**.
- `change-scope` (base `c389f96b...`, head `2d2307e4...`) reports only:
  - `packages/subprocess/subprocess-local/src/spawn.ts`
  - `packages/subprocess/subprocess-local/tests/spawn.spec.ts`

### Severity / confidence
- **INT-BL-01 (P1, High):** Branch is a narrow patch on an old baseline; integration readiness depends on rebasing/cherry-picking retained intent onto a modern upstream base, not merging this branch directly.

## 2) Build/dependency/release/CI gate audit

## 2.1 Manifest/export/source-built entrypoint inventory
- Workspace manifests scanned: **269** (`packages/*/*`, `apps/*`, root).
- Package manifests missing `exports`: **0**.
- Package manifests missing both `types` and `bin`: **0**.
- Key supported entrypoints verified in manifests:
  - CLI shipped entrypoint: `apps/cli/package.json` -> `bin.dsh = lib/bin.js`.
  - Source launch lane (dev only): root script `dsh = node --import tsx/esm apps/cli/src/bin.ts`.
  - SDK TS public entrypoints: `@deepseek-ai/dsh-sdk-client`, `@deepseek-ai/dsh-sdk-protocol`, `@deepseek-ai/dsh-sdk-jsonrpc-server` all export built `lib/*`.
  - Python lane entrypoints: `python/sdk-runtime/pyproject.toml` exposes script `dsh = deepseek_harness_runtime:main`; `python/sdk/pyproject.toml` depends on `deepseek-harness-runtime-bin`.

### Severity / confidence
- **BLD-01 (Info, High):** Manifest/export consistency is structurally strong in this checkout.

## 2.2 CI workflow event/runners/trust boundaries
- `ci.yml` is PR-triggered and uses custom hosted pools by default (`dsh-ubuntu-24-04-16core`, `dsh-windows-2025-16core`) with optional failover to self-hosted via repository variables.
- `ci-master.yml` is `push master` + manual, with explicit self-hosted standby drills (`[self-hosted, linux, x64, vm-backup]`, `[self-hosted, dsh-win-ci, windows]`) and disabled `serial-macos` (`if: false` TODO).
- `release.yml` and `release-publish.yml` separate rehearsal and publish, and publish is manual-only with environment gates.
- `python-release.yml` is manual dispatch with explicit publish authorization checks and OIDC PyPI jobs.

### Severity / confidence
- **CI-01 (P1, Medium):** Fork CI readiness is unproven until runner entitlement and required-check rules are verified on the actual integration PR (workflow YAML alone cannot prove branch-protection wiring).
- **CI-02 (P2, Medium):** Self-hosted trust gating is lane-asymmetric (`node-compat` has stricter fork checks than several other lanes under failover conditions).

## 2.3 Actual CI/run evidence (this audit)
- `actions_list(list_workflow_runs)` and `get_job_logs` executed.
- Branch filter for `spill-dir-deletion-guard` returned **0 workflow runs**.
- Recent runs are mostly dynamic Copilot agent workflows, not full CI lanes.
- Failed run `37695087569` log confirms quota termination (`You have exceeded your monthly quota`) and records model string `gpt-5.3-codex`.

### Severity / confidence
- **CI-03 (P1, High):** No full CI evidence exists for the target branch in this fork at this time.

## 2.4 Release/session version records and native evidence
- Current checkout `packages/core/session/src/types.ts` declares `SESSION_FORMAT_VERSION = 2`.
- Upstream pinned commit `5badb150...` corresponds to prerelease `dsh-v0.2.1-alpha.1` (verified via release API).
- Prior Codex claim about a `docs/session-format-status.md` mismatch could not be revalidated in this checkout because that path is absent here.
- Native Windows evidence in this audit is configuration-level only; no native Windows test execution was performed.

### Severity / confidence
- **REL-01 (P2, Medium):** Session/release assertions must be evaluated against the chosen integration baseline; historical report paths not present in current checkout need re-verification before acting.

## 3) Completeness inventory: TODO/FIXME/XXX, stubs, placeholders

### 3.1 Marker inventory (this run)
- Repo-wide `TODO|FIXME|XXX` lexical count: **149 matches / 100 files** (includes docs/vendor/tests).
- Source-focused (`packages/**/src`, `apps/**/src`, `native/**/src`): **53 matches / 37 files**.
- Major clusters include: hooks lifecycle TODOs, settings redaction/quiescence TODOs, E2B deferred TODOs, shell/background outcome TODOs, terminal PTY lifecycle TODOs.

### 3.2 Not-implemented/placeholder/stub lexical inventory
- Broad lexical scan (`not implemented|placeholder|stub...`) is heavily noisy (UI placeholders, test runtime scaffolding, intentional compatibility shims).
- Intentional limits explicitly documented in package READMEs under `## Known Limitations and Deferred Work` for:
  - MCP (`packages/mcp/mcp-client`),
  - Workflow (`packages/workflow/workflow*`, `tool-workflow`),
  - Subagent (`packages/subagent/*`),
  - Sandbox (`packages/sandbox/*`),
  - LLM (`packages/llm/*`).

### Severity / confidence
- **CMP-01 (P2, High):** Marker presence alone is not proof of broken behavior; triage must treat documented intentional limits separately from regressions.

## 4) LLM/MCP/workflow/subagent/sandbox/desktop coverage gaps (confirmed)

- **LLM:** core service intentionally excludes retry/caching/rate-limit execution; adapter-level limitations (`tool_choice`, shared HTTP service adoption, image/output constraints).
- **MCP:** tools-only bridge; resources/prompts and task-based extension remain deferred.
- **Workflow:** no background start/poll/resume; worker-thread VM is not a security boundary.
- **Subagent:** ACP one-shot/traceability gaps, parent-liveness constraints, no durable cross-process mailbox.
- **Sandbox:** seam only expresses filesystem policy; local provider documents partial enforcement conditions.
- **Desktop:** substantial operational complexity and platform constraints documented; Linux desktop release target explicitly unsupported.

### Severity / confidence
- **CAP-01 (P2, High):** These are known, declared capability boundaries; they are not new defects but must be explicit in merge-readiness criteria.

## 5) Codex build/integration/marker report verification and corrections (PR #1)

### Confirmed/high-signal points
- Baseline staleness and one-commit custom scope are correct.
- Upstream spill refactor supersession risk is correct: upstream owns collector behavior in `src/output.ts` with expanded tests.
- Need for baseline-first integration lanes is correct.

### Corrections/qualification
- **BUILD-01 runner entitlement:** keep as conditional risk; do **not** claim lack of entitlement without live branch-protection/runner policy evidence.
- **Marker report completeness:** lexical inventory is useful but cannot establish semantic completeness; broad stub/placeholder scans include large intentional noise.
- **Session-format record mismatch claim:** requires baseline-specific re-validation before actioning in this fork checkout.

## 6) Baseline-first disjoint implementation lanes and merge gates

## 6.1 Parallel ownership lanes (disjoint)
1. **Lane A — Baseline integration (Owner: integration maintainer)**
   - Create integration branch from pinned upstream commit (`5badb150...` or newer approved pin).
   - Reapply only retained custom requirement(s) after upstream diff review.
2. **Lane B — Subprocess spill intent reconciliation (Owner: subprocess maintainer)**
   - Decide whether directory recreation semantics are still required.
   - If yes, implement on upstream `src/output.ts` ownership model with tests; if no, retire redundant behavior and keep upstream path.
3. **Lane C — CI/runners and trust policy (Owner: infra maintainer)**
   - Verify required checks and runner entitlement on the real integration PR.
   - Normalize failover trust conditions across lanes.
4. **Lane D — Marker triage (Owner: area maintainers by package)**
   - Convert high-risk TODO/FIXME items into tracked issues/PR tasks, preserving intentional limits as documented constraints.
5. **Lane E — Windows/native evidence (Owner: windows maintainer)**
   - Run required Windows-native gates on integrated head; record pass/fail evidence.

## 6.2 Dependency order
- A blocks B/C/D/E (must run on integrated baseline).
- C and E block final merge gate.
- B and D can run in parallel after A.

## 6.3 Exact merge gates (must all pass)
1. Integrated branch pinned and documented (`base`, `head`, merge-base).
2. `change-scope` regenerated for integrated head and used to select focused local checks.
3. Subprocess retained-intent decision recorded with focused regression evidence.
4. Required CI checks green on exact integrated SHA (including Windows-required lanes).
5. No unresolved P1/P2 findings without explicit acceptance rationale.
6. PR #1 (tracking) updated with final report link, evidence, and open risks.

## 7) Commands executed and results (audit evidence)

- `git rev-parse HEAD` -> `2d2307e4d0f43bac355ff83814e9f921482401e5`.
- `git fetch --all --tags --prune` + branch listing -> only origin copilot branch fetched.
- `corepack pnpm --silent run change-scope --base c389f96b... --head 2d2307e4...` -> two committed files in scope.
- `git fetch https://github.com/deepseek-ai/deepseek-harness.git 5bad...` + `git rev-list --left-right --count` -> `1 20734` vs upstream, `0 20734` for fork master vs upstream.
- `git diff --name-status c389f96b..5bad... -- packages/subprocess/subprocess-local` -> confirms upstream refactor including new `src/output.ts` and expanded tests.
- `actions_list(list_workflow_runs)` + `actions_get(get_workflow_run)` + `get_job_logs(failed_only)` executed; quota-failure logs captured for `run_id=37695087569`.
- `actions_list(list_workflow_runs branch=spill-dir-deletion-guard)` -> `total_count: 0`.
- Multiple `rg/glob/view` inventories over workflows, manifests, README limitations, and marker scans.

## 8) Coverage limitations of this audit

- This is an analysis-only/report-only pass; no production code changes, dependency upgrades, releases, merges, settings/permission changes, or default-branch changes were made.
- No full-suite local runs were executed.
- Lexical inventories are not semantic correctness proofs.
- Branch-protection requirements, runner entitlements, and environment approvals require live repository settings checks on the integration PR.

