---
name: delivery-audit
description: Adversarially audit any unit of work — an experiment, an analysis, a delivery — by decomposing its reasoning chain into typed nodes and auditing each node against raw evidence, fanned out to parallel independent reviewers (Codex, Claude, or harness subagents). Use before delivering non-trivial work, when a number or result is reported, or when the user asks to review, double-check, 审核, 复核, or 质疑 a delivery.
---

# Delivery Audit

Stance: the delivery is wrong until proven right. The user is careless, and the executing model is trained to agree with the user and with itself. Neither the transcript's claims nor the user's premises count as evidence. Only fresh, reproduced observations count.

Most real failures are three kinds — hunt these before anything else:

1. **臆想 (assumption stated as fact)** — "should", "normally", "it prints 0 when done" — never checked.
2. **口径错误 (definition mismatch)** — the statistic does not mean what the report claims. Example: the progress counter reads 0 because every job crashed, not because the run finished. Every number's semantics must be traced to the code that produces it.
3. **没看原始证据 (summary over source)** — conclusions drawn from dashboards and aggregates while the trace/log/rows that would refute them sit unopened.

When the user reports a result of their own, audit it with the same rigor; a user's number is a claim, not a fact.

## Scope

Run the full procedure before delivering non-trivial work and on demand. For a trivial one-step answer, audit only the claims directly.

## Procedure

### 1. Reconstruct the ask

Re-read the user's original messages — not your summary of them. List every explicit requirement, constraint, and acceptance criterion verbatim. Check for silent substitution: an easier or more familiar problem solved instead of the asked one. Any requirement without evidence is NOT MET.

### 2. Inventory the claims

Write down every claim the delivery makes: files changed, behaviors fixed, numbers, scores, "tests pass", "X works". Each claim is a defendant. Every one ends the audit as VERIFIED (with fresh evidence), REFUTED, or UNVERIFIED.

### 3. Decompose the reasoning chain

Rebuild the whole task as a numbered chain of typed nodes — one node per step of reasoning, chronological, from premise to final claim. Each node records:

- **type**: `premise` / `observation` / `computation` (any statistic or aggregate) / `interpretation` / `decision` / `delivery-claim`
- **statement**: what was concluded
- **evidence pointer**: the file, command, or artifact it rests on
- **question**: what a skeptic would ask first

The chain is the audit's backbone and goes into the audit packet. Note: the chain is built by the defendant (the executing model), so step 5 and the reviewers are required to hunt for nodes it omits.

### 4. Audit each node

For every node, in order:

- **Definition (口径)**: what exactly does this count or assert? Trace it to the producing code or query (a ≤5-line snippet or one command). State the actual definition in one line. Does it answer the ask from step 1?
- **Evidence**: open the raw artifact — trace, log, rows — not the summary that cites it. Re-derive the value now.
- **Assumption (臆想)**: mark every unverified premise inside the node. Check it, or flag the node UNVERIFIED.
- **Semantics probe** for every aggregate/counter: pick one concrete succeeding unit and one concrete failing unit from raw output, and confirm the aggregate counts each correctly. A counter at 0 must be shown to mean "done", not "dead".

Node verdict: VERIFIED / REFUTED / UNVERIFIED.

### 5. Hunt for missing nodes

Grep the session transcript for errors, retries, warnings, and commands that produced no node in the chain — a swallowed failure is a finding. Grep for "should", "probably", "looks right", "正常", "按理" — each hit is an unchecked assumption.

### 6. Global checks

- **Rule compliance**: read the project's rule files (e.g. `AGENTS.md`); extract every MUST/NEVER this task touched; verify each concretely (immutable inputs show an empty `git diff`, required checks actually ran).
- **Completeness**: diff contains no stubs, TODOs, debug leftovers; every removed/renamed symbol's callers migrated (LSP references, not memory); no test deleted, weakened, or re-pinned; delivered scope matches step 1 — no silent narrowing, no invented extra scope.
- **Real surface**: run the deliverable the way its consumer will — invoke the binary/CLI, drive the UI, call the library from a real caller. A passing test suite is not proof the user-facing path works.
- **Falsification**: bounded effort to break the result — boundary inputs, empty/adversarial cases, the opposite configuration. One concrete counterexample outweighs any number of passing happy paths.

### 7. Fan out parallel independent reviewers

Partition the chain and global checks into independent bundles (typical: claims recomputation / 口径 audit / missing-nodes & process / rules & completeness). Spawn one fresh reviewer per bundle in parallel — one batch, not serialized. Prefer mixing model families; same-family reviewers share blind spots:

- `codex exec --ephemeral --sandbox workspace-write -C <project-root> "<instructions>"`
- `claude -p "<instructions>"` (if installed)
- harness `reviewer` subagents via the task tool

Reviewers must be able to run real commands, including ones that write logs and results — a read-only sandbox makes verification theatrical. Forbid source and config edits in the prompt instead, snapshot `git status --porcelain` before and after, and treat any unexpected source change as a finding.

Each reviewer gets the audit packet — a scratch file such as `/tmp/delivery-audit-<ts>.md` containing facts and pointers only:

```text
# Audit packet
- Original request: <the user's first message and later amendments, verbatim>
- Claims: <numbered list>
- Reasoning chain: <the typed nodes from step 3>
- Your bundle: <which nodes/checks this reviewer audits>
- Project root: <path>            # reviewers read AGENTS.md themselves
- Changed files: <git status --porcelain output>
- Reproduce claim N: <one exact command per claim>
- Artifacts: <result/fact/data paths>
- Session transcript: <~/.omp/agent/sessions/<project>/<ts>.jsonl;
  grep it for errors and unreported failures — never read it linearly>
```

Never hand reviewers a prose summary of the session: the summarizer is the defendant and will omit its own failures. Never make the transcript the primary input either: it anchors the reviewer on the executor's self-narrative and burns its budget on reading instead of verifying.

Required reviewer report format: numbered findings with severity (BLOCKER/MAJOR/MINOR) and reproducible evidence; per-node verdicts; verdict DELIVER / DO NOT DELIVER.

### 8. Verify findings, fix, re-audit

Reproduce every returned finding yourself before accepting it — reviewers hallucinate too. Fix valid findings. A fix that changes code, data, metrics, or claims sends the affected bundles back for re-audit. Cap the loop at three rounds.

**Source changes need prior approval.** Before modifying any source code — tracked or untracked: harness, scripts, configs, tests — report the finding and the proposed patch to the user and wait for explicit approval. Applying a source patch and then reporting it is a violation, however small the patch. Live data/state repairs that follow an already-documented precedent in the repo (e.g. normalizing a state file the same way a previous recovery note records) may be applied immediately to stop an armed failure, but must be reported at once with before/after evidence.

## Verdict and report

Report in this order:

1. **Findings**: numbered, severity (BLOCKER / MAJOR / MINOR), concrete evidence (command + output, `file:line`), required fix.
2. **Chain table**: every node → VERIFIED / REFUTED / UNVERIFIED, with the 口径 line for each computation node.
3. **Claims table**: every claim from step 2 → VERIFIED (with fresh evidence) / REFUTED / UNVERIFIED.
4. **Verdict**: `DO NOT DELIVER` while any BLOCKER stands or any load-bearing claim is UNVERIFIED; otherwise `DELIVER`, listing residual MINOR issues.

Never soften a finding because the overall result looks good. A correct delivery reported with a false supporting claim is still misreporting.

## Anti-patterns

- Reading the dashboard instead of the trace; treating "no output" or "counter is 0" as success.
- Citing "tests pass" from memory or from before the last edit.
- Verifying a fix by re-reading the diff instead of re-running the reproduction.
- Accepting the user's or the transcript's number without recomputation.
- Auditing only the artifact and not whether it answers the original ask.
- Rounding an incomplete run up to a complete one.
- Downgrading a finding because fixing it is inconvenient.
