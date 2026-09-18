# Creative tools: focused checks, 2026-09-18

Shared-space baseline: `8c0b0af9d54b3c766a68303afa2d8bc40d529283`.
All four source working trees were clean before these checks.
Each command ran once in its named repository using its existing `.venv` on macOS.
All commands exited 0. Output below is the pytest summary, with progress indicators omitted.
No live provider evaluation or conformance run was performed. These passing tests are
OBSERVED behavior, not MEASURED A/B evidence. Test durations are runner output, not benchmarks.

To reproduce, use the pinned repository revision and its AGENTS.md development setup;
the commands assume that setup is already present. No fresh-install claim is made.

## possibly

**Evidence: OBSERVED** — one focused fixture-based test execution.

Repository: https://github.com/robotdad/possibly

Commit: `28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b`; local branch: `feature/a2ui-review-workspace`.

```sh
.venv/bin/python -m pytest -q tests/test_contracts.py tests/test_fanout.py tests/test_performance.py -k 'question_answer or retry_and_one_use or review_drafts or late_generation or process_batch_divides or streamed_candidates_survive or deterministic_finalize or finalize_rejects'
```

```text
8 passed, 34 deselected in 2.95s
```

## amplifier-smart-tool-unfold

**Evidence: OBSERVED** — one focused fixture-based test execution.

Repository: https://github.com/robotdad/amplifier-smart-tool-unfold

Commit: `8fb14bc81e6d0f892963832c53aa598401b19370`; local branch: `main`.

```sh
.venv/bin/python -m pytest -q tests/test_workflows.py -k 'pack_roundtrip or invalid_pack or review_authority or versions_and_omissions'
```

```text
4 passed, 5 deselected in 0.68s
```

## amplifier-smart-tool-outtake

**Evidence: OBSERVED** — one focused fixture-based test execution.

Repository: https://github.com/robotdad/amplifier-smart-tool-outtake

Commit: `d441a37faea2a7fb762f2ca0395709ff1b542ab3`; local branch: `main`.

```sh
.venv/bin/python -m pytest -q tests/test_intelligence.py -k 'disclosure_scope or cannot_submit or model_limit or text_budget or provider_gate'
```

```text
5 passed, 10 deselected in 1.53s
```

## amplifier-smart-tool-stories

**Evidence: OBSERVED** — one focused fixture-based test execution.

Repository: https://github.com/robotdad/amplifier-smart-tool-stories

Commit: `3b82a038c5aa321a126a6243fdb00b4168e0e7c2`; local branch: `main`.

```sh
.venv/bin/python -m pytest -q tests/test_contract_gaps.py tests/test_narration.py tests/test_media.py -k 'initial_question or source_status or calculation_requires or acceptance_is or partial_failure_uncertainty or reuse_notes or only_changed or zip_rejects'
```

```text
8 passed, 21 deselected in 1.62s
```
