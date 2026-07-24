---
name: impl-peer-review-memory
description: >-
  Cross-LLM implementation peer review with optional long-term correction memory. Use after an
  implementation pass when the user explicitly requests an external diff review and lesson-memory
  may capture human corrections. Don't use for spec or TechSpec review.
trigger: explicit
disable-model-invocation: true
argument-hint: "[--files p1,p2] [--context p1,p2] [--base ref] [--staged] [--out dir] [--verify cmd] [--ide <ide>] [--model <model>] [--reasoning <effort>]"
metadata:
  author: Vinícius Bôscoa
  github: https://github.com/virb30
  repository: https://github.com/virb30/ia-resources/skills
  based-on: https://github.com/pedronauck/skills/tree/main/skills/mine/impl-peer-review
  related-skills: impl-peer-review, lesson-memory
---

# Implementation Peer Review with Memory

An independent reviewer runtime pressure-tests an implementation diff. This skill runs that
cross-LLM review only on explicit user request after an implementation pass; its scope is the diff
plus any `--context` files the user names, decoupled from any task-tracking system. **Compozy is
the review engine**: each round is one `compozy exec` against a configurable runtime. The findings
file the reviewer writes is the sole source of truth — treat `compozy exec` stdout/stderr and the
event log as operational evidence only.

## Bundled files

Resolve every `scripts/<name>` or `references/<name>` path relative to this `SKILL.md`'s directory
(`<skill-dir>`). All bundled files are read-only — read or run them, never edit them during a round.

## Asking the user

When a step tells you to ask the user (which findings to incorporate, whether to run another
round), use the runtime's interactive question tool — the one that presents a question and pauses
until the user answers (e.g. AskUserQuestion). If the runtime has no such tool, present the
question as the complete assistant message and stop; answer it yourself only after the user replies.

At the start of the run, detect whether `lesson-memory` is available in the runtime's skill catalog
or at `<skill-dir>/../lesson-memory/SKILL.md`. When available, run the correction-memory protocol
after either event:

- the workflow needs a user decision, clarification, correction, or missing input before it can
  continue;
- the user corrects any review finding, rationale, remediation, assumption, or procedure, including
  an unsolicited correction.

### Correction-memory protocol

1. Finish the action directly implied by the user's intervention, then ask whether the user wants
   to record the correction in long-term project memory.
2. When the user declines, resume the review without writing memory.
3. When the user accepts, derive exactly three facts from the conversation:
   - what was wrong;
   - why it was wrong;
   - the correct approach.
4. Ask a targeted follow-up for each fact that cannot be established from context. Treat these
   follow-ups as part of the same protocol rather than new correction events.
5. Start one subagent with the three facts verbatim. Instruct it to read the available
   `lesson-memory` skill in full and follow it to persist the correction. Instruct it to return
   lesson/index paths on success or collision details when a user decision is required. Give the
   subagent no review implementation task.
6. Wait for the subagent. When it reports a collision, ask the user to overwrite or abandon, send
   that decision to the same subagent, and wait for its final result.
7. Report the subagent's memory result and resume the review.

*Done when:* the intervention has either declined memory capture or produced a completed subagent
result from three concrete facts.

If `lesson-memory` is absent, continue the review without offering memory capture. If the runtime
cannot start a subagent after the user accepts, report that memory capture could not run and resume
the review without writing memory in the parent agent.

## Inputs

All inputs are optional. Defaults make the common path `impl-peer-review` with no arguments.

- `--files <p1,p2,...>` — scope the review to explicit paths instead of the full branch diff.
- `--context <p1,p2,...>` — additional context files to feed the reviewer (a spec, ADR, design
  doc, RFC, README). The skill never assumes any of these exist.
- `--base <git-ref>` — base ref for the diff. Defaults to `main`. Use `--base HEAD~N` for a
  narrower scope.
- `--staged` — review staged changes only (`git diff --staged`) instead of a branch diff.
- `--out <dir>` — output directory for round artifacts. See Output Directory.
- `--verify <cmd>` — the verification command (build/tests) used as both the readiness gate and
  the post-remediation gate. When omitted, auto-detect per `references/readiness-checks.md`; if
  none resolves, ask the user.
- `--ide <ide>` (default `claude`): reviewer runtime, forwarded to `compozy exec --ide`. Accepted
  values mirror `compozy exec`: `codex`, `claude`, `cursor-agent`, `droid`, `opencode`, `pi`,
  `gemini`, `copilot`. On an invalid value, list the accepted values and ask the user to choose —
  do not fall back to `claude`.
- `--model <name>` (default `opus`): forwarded to `compozy exec --model`. Do not pre-validate; let
  `compozy` surface incompatibilities (see Step 3 for the stale-model response).
- `--reasoning <effort>` (default `xhigh`): forwarded to `compozy exec --reasoning-effort`.
  Accepted: `low`, `medium`, `high`, `xhigh`.

## Output Directory

Resolve `<out>` in this order:

1. `--out` if provided.
2. Otherwise, if the repo exposes an unambiguous `.compozy/tasks/<slug>/` layout for the current
   work, use that task's `qa/` directory: `.compozy/tasks/<slug>/qa/`.
3. Otherwise, default to `.peer-reviews/<UTC-timestamp-YYYYMMDDTHHMMSSZ>/` at the repository root.

Create the directory if it does not exist. Every artifact is versioned with `-roundN`; never
overwrite a prior round — increment to the next round instead.

## Findings Artifact

Each round has exactly one authoritative findings file: `<out>/impl-review-findings-roundN.md`.
The reviewer writes that file and no other. If the target path is missing, ambiguous, unwritable,
or outside `<out>`, the reviewer refuses and stops — no stdout fallback.

The findings format — frontmatter, section headings, and per-item fields — is defined in
`references/impl-review-prompt.md` and enforced by `scripts/validate-findings.sh`.

## Procedures

**Step 0: Verify Compozy is available**

Confirm the `compozy` binary is on `PATH` (e.g. `command -v compozy`). If missing, abort with a
one-line message to install Compozy; the cross-LLM independence is the point, so require it rather
than falling back to a harness-native subagent. This skill calls `compozy exec` directly — no
`--agent` install is required.

*Done when:* `compozy` resolves on `PATH`, or the review has stopped with the installation message.

**Step 1: Validate Input and Compute Scope**

1. Confirm the user has just completed (or paused) the implementation pass and explicitly asked to
   review the current state. Do not run review rounds during active editing.
2. Resolve the diff scope:
   - If `--files` is provided, verify each path exists and limit the diff to those paths.
   - If `--staged` is provided, use `git diff --staged`.
   - Otherwise run `git diff <base>...HEAD --name-only` (default `<base>` is `main`) to compute the
     changed file set. If the diff is empty, abort and tell the user there is nothing to review.
3. Resolve `<out>` (see Output Directory) and create it if needed.
4. Resolve the verification command (`--verify`, else auto-detect, else ask the user).
5. **STOP. Read `references/readiness-checks.md` in full before running the review.** Verify every
   readiness marker passes; the markers and their auto-detect order live in that file. If any
   marker fails, report the failed markers and abort — external review on a broken or incomplete
   change wastes credit and produces noise.
6. Determine the next review round number by listing existing
   `<out>/impl-review-findings-round*.md` and `<out>/impl-review-summary-round*.md` files. Start at
   `round1` when none exist.

*Done when:* the diff scope, output directory, verification command, readiness result, and next
round number are resolved.

**Step 2: Compose the Review Prompt**

1. **STOP. Read `references/impl-review-prompt.md` in full before composing the prompt.** It is the
   canonical executable reviewer prompt template; substitute its placeholders verbatim and do not
   paraphrase it. The assembled prompt must start with the reviewer instructions, not with a
   Markdown wrapper describing the template.
2. Capture the diff payload:
   - Run `git diff <base>...HEAD -- <changed-files>` (or `git diff --staged -- <changed-files>` for
     `--staged`) and write the raw patch to `<out>/impl-review-diff-roundN.patch`.
   - Run `git log --oneline <base>...HEAD -- <changed-files>` and capture the commit list (empty
     string if `--staged`).
3. Define the round artifact paths under `<out>/`: `impl-review-findings-roundN.md`,
   `impl-review-events-roundN.jsonl` (event log), `impl-review-result-roundN.err` (stderr),
   `impl-review-status-before-roundN.txt`, `impl-review-status-after-roundN.txt`, and
   `impl-review-validation-error-roundN.md` (only when needed).
4. Discover existing project rule files to populate `{project_rules}`: root-level `CLAUDE.md`,
   `AGENTS.md`, `.cursor/rules/*`, `.cursorrules`, `CONTRIBUTING.md`; nested `CLAUDE.md`/`AGENTS.md`
   in the directories the diff touches; and any project memory/directive docs (e.g. `docs/_memory/`,
   standing directives, lessons indexes). Nested and memory rules hold the load-bearing invariants
   root-only discovery misses.
5. Substitute the placeholders in the prompt template:
   - `{scope_summary}` — one-paragraph description of what was implemented. Derive from the user's
     brief, the commit messages, or — if `--context` was passed — the linked spec/PRD.
   - `{context_paths}` — newline-separated repo-root paths from `--context`, or `none`. When the
     reviewed diff implements a task from a spec directory (e.g. `.compozy/tasks/<slug>/`,
     `specs/<name>/`, `docs/rfcs/<name>/`), also include that spec's contract-bearing sibling
     artifacts here even when the user passed no `--context` (see Guardrails: Contract parity).
   - `{project_rules}` — newline-separated discovered rule-file paths, or `none`.
   - `{changed_files}` — newline-separated repo-root paths.
   - `{diff_path}` — repo-root path to the patch file from step 2.
   - `{commit_list}` — captured `git log --oneline` output, or `none` if `--staged`.
   - `{findings_path}` — exact absolute path to `<out>/impl-review-findings-roundN.md`.
   - `{round}` — numeric review round `N`.
6. Write the assembled prompt to `<out>/impl-review-prompt-roundN.md`.

*Done when:* the raw patch and fully substituted prompt exist at their round-specific paths.

**Step 3: Execute the Cross-LLM Review**

1. Capture the pre-run status snapshot:

   ```bash
   git status --short > <out>/impl-review-status-before-roundN.txt
   ```

2. Run (substitute the resolved `--ide`/`--model`/`--reasoning`, defaults `claude`/`opus`/`xhigh`):

   ```bash
   compozy exec --ide <ide> --model <model> --reasoning-effort <reasoning> --format json --prompt-file <out>/impl-review-prompt-roundN.md > <out>/impl-review-events-roundN.jsonl 2> <out>/impl-review-result-roundN.err
   ```

3. Capture the post-run status snapshot:

   ```bash
   git status --short > <out>/impl-review-status-after-roundN.txt
   ```

4. If the command exits non-zero, fail loudly — no silent retry. If stderr shows
   `The model 'X' does not exist`, the runtime may be set to a stale name: surface the configured
   model and confirm with the user before rerunning; do not substitute a model yourself.
5. Require the findings target file to exist after the command exits. If missing, the round is
   invalid even when `compozy exec` exited 0 — write `impl-review-validation-error-roundN.md` and
   ask whether to rerun.

*Done when:* the command, status snapshots, and findings-presence check have completed with any
invalid state surfaced to the user.

**Step 4: Validate and Summarize Findings**

1. Run the bundled read-only validator:

   ```bash
   bash <skill-dir>/scripts/validate-findings.sh --kind implementation --round N --path <out>/impl-review-findings-roundN.md
   ```

2. Manually inspect the findings file and verify the semantic contract:
   - every finding has a real file path/line or an explicit reason it is not applicable;
   - blockers include a rationale tied to project rules or architecture constraints;
   - no `TBD`, placeholder text, invented paths, or stdout-only findings;
   - when spec contract artifacts were passed in `{context_paths}`, the findings explicitly assess
     contract parity field by field (see Guardrails: Contract parity) — a `SHIP` verdict with no
     contract-parity assessment is an invalid round;
   - comparing the pre/post status snapshots shows no changes outside the expected review
     artifact/log paths.
3. If validation fails, write `<out>/impl-review-validation-error-roundN.md` with the failed
   checks, command, exit status, and artifact paths. Do not summarize the round as `SHIP`.
4. Write `<out>/impl-review-summary-roundN.md` from the validated findings file with: the verdict
   (`SHIP` / `FIX_BEFORE_SHIP` / `REWORK`); one-line rationale per blocker; the risks list; the
   nits list; files most likely affected by remediation; and the operational artifact paths.
5. Present a concise user-facing summary: verdict, blocker/risk/nit counts, main themes, and the
   artifact paths written for the round.
6. Hold all source, test, config, doc, and commit edits until Step 5.

*Done when:* the findings are validated, the summary artifact is written, and the user has received
the verdict and artifact paths.

**Step 5: User-Directed Remediation**

1. Ask the user which findings to incorporate:
   - A) all blockers
   - B) selected blockers/risks/nits
   - C) nothing — keep the review as a record only
   - D) manual edits before any remediation
2. Apply only the findings the user selected.
3. Re-run the resolved verification command after applying any code change. If it fails, fix the
   new failure or surface it back to the user before declaring remediation done.
4. Record the remediation decision in `<out>/impl-review-remediation-roundN.md`, listing:
   incorporated items with the new commit/diff range; deferred items; files changed; and the
   verification command and outcome with a timestamp.
5. Show the user what changed and what remains deferred. Commit or push only on explicit user
   instruction.

*Done when:* the selected remediation is applied and verified, its decision artifact is written,
and deferred work is reported.

**Step 6: Optional Additional Rounds**

1. Ask whether the user wants another peer-review round against the updated code or wants to stop
   with the current state.
2. Run further rounds only when the user explicitly requests one: re-run from Step 2 against the
   new diff and create a fresh `roundN+1` artifact set in the same `<out>` directory.

*Done when:* the user has stopped or every explicitly requested additional round has completed.

## Guardrails

**Contract parity.** For a spec-workflow diff, engineering quality alone can never earn `SHIP`.
When the diff implements a task from a spec directory (`.compozy/tasks/<slug>/`, `specs/<name>/`,
`docs/rfcs/<name>/`), the reviewer must receive that spec's contract-bearing sibling artifacts —
canonical example documents, input/schema tables, QA seeds, test contracts, parity maps — and
assess the deliverable against them field by field. A round whose `{context_paths}` omitted those
artifacts is invalid; a task-file paraphrase is never a sufficient contract source. Real incident:
seven rounds reached `SHIP` on a deliverable that contradicted the spec's canonical example document
because no round ever received it.

**Credit once.** `compozy exec` is the only place this skill spends external review credit. Run it
once per round; rerun only when the round is explicitly invalid and the user asks for it.
