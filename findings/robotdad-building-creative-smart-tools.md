# Findings: continuing, reviewing and handing off creative smart tools

**Who:** robotdad
**What this came from:** [Possibly](https://github.com/robotdad/possibly), [Unfold](https://github.com/robotdad/amplifier-smart-tool-unfold), [Outtake](https://github.com/robotdad/amplifier-smart-tool-outtake), and [Stories](https://github.com/robotdad/amplifier-smart-tool-stories).
**Status:** Contribution draft, 2026-09-18. Source inspection plus 25 focused passing tests. No live provider experiment, conformance run, or quality benchmark was performed for this review. Tests use fixtures and injected/scripted intelligence. Exact revisions, commands and results are in [the evidence record](../evidence/robotdad/creative-tools-focused-checks.md). Possibly was inspected on a feature branch; this is evidence about the pinned implementation, not a claim about its default branch.

## 1. Continuing a smart call has two working shapes

**Evidence: OBSERVED** — two implementations and two passing clarification tests; no comparative usability measurement.

[Open question 5](../open-questions.md#roadmap-questions-we-did-not-touch) says the existing tools never needed continuation. Ours do. A design or story request can be incomplete, and the tool must preserve the question and the work it belongs to while waiting for the caller.

Possibly answers a retained question and requeues the **same operation**, rechecking its existing grant. Stories creates a **child operation**, records the prior question and answer, retains the story and provider, and requires a new bounded grant. Both replay an identical answer request and reject a conflicting second answer.

Evidence: Possibly [answer implementation](https://github.com/robotdad/possibly/blob/28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b/src/possibly/lib.py#L616) and [test](https://github.com/robotdad/possibly/blob/28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b/tests/test_contracts.py#L114); Stories [implementation](https://github.com/robotdad/amplifier-smart-tool-stories/blob/3b82a038c5aa321a126a6243fdb00b4168e0e7c2/src/amplifier_smart_tool_stories/lib.py#L408) and [test](https://github.com/robotdad/amplifier-smart-tool-stories/blob/3b82a038c5aa321a126a6243fdb00b4168e0e7c2/tests/test_contract_gaps.py#L14).

**Lesson:** the useful common contract is correlation, retained context, replay/conflict semantics and explicit budget behavior. These examples do not justify standardizing a single session mechanism. They extend the existing long-running proposal: `needs_input` is neither dead nor actively computing.

## 2. Human review is retained state with authority boundaries

**Evidence: OBSERVED** — fixture tests for drafts, stale results, bounded review execution and revision-specific acceptance.

A caller and a person in a dashboard can act on the same artifact at different times. Possibly retains unsent drafts separately from decisions, ignores an older draft sequence, and marks a late model result superseded after a brief correction. Unfold refuses review generation without an allowance, replays an identical submission without spawning another worker, and rejects a stale revision. Stories records acceptance against an exact revision without rewriting the artifact or its checks.

Evidence: Possibly [draft test](https://github.com/robotdad/possibly/blob/28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b/tests/test_contracts.py#L411) and [late-result test](https://github.com/robotdad/possibly/blob/28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b/tests/test_contracts.py#L169); Unfold [review test](https://github.com/robotdad/amplifier-smart-tool-unfold/blob/8fb14bc81e6d0f892963832c53aa598401b19370/tests/test_workflows.py#L84); Stories [acceptance test](https://github.com/robotdad/amplifier-smart-tool-stories/blob/3b82a038c5aa321a126a6243fdb00b4168e0e7c2/tests/test_contract_gaps.py#L114).

**Lesson:** feedback, unsent drafts, permission to generate, and human acceptance are separate facts. An interactive tool needs to identify the revision each concerns. A dashboard is an adapter to those facts, not an independent owner of business logic. These tests establish state behavior, not whether people understand the UI.

## 3. Recovery must distinguish replay, finalization and another paid attempt

**Evidence: OBSERVED** — deterministic recovery and simulated uncertain speech completion; no real provider billing experiment.

Possibly can fail after retaining complete checked output. Its `finalize_operation` publishes that output with intelligence removed; incomplete evidence or a changed brief is rejected. Stories retains completed speech clips, reuses unchanged clips, and generates only changed narration. An uncertain speech attempt blocks a blind retry, even under a new request ID. Retrying uncertain work requires `retry_uncertain=true` on a new request.

Evidence: Possibly [finalization tests](https://github.com/robotdad/possibly/blob/28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b/tests/test_performance.py#L97); Stories [reuse and failure tests](https://github.com/robotdad/amplifier-smart-tool-stories/blob/3b82a038c5aa321a126a6243fdb00b4168e0e7c2/tests/test_narration.py#L48) and [uncertain-attempt guard](https://github.com/robotdad/amplifier-smart-tool-stories/blob/3b82a038c5aa321a126a6243fdb00b4168e0e7c2/src/amplifier_smart_tool_stories/speech.py#L382).

**Lesson:** “retry” is too ambiguous for a caller spending someone else's budget. Replaying a receipt, publishing already validated work, and making another external request need distinct semantics. This extends the [long-running cost proposal](../proposals/long-running-is-the-hosts-problem-too.md): a client-side failure does not prove the provider did no billable work. Local idempotency does not guarantee exactly-once external execution.

## 4. Budgets must survive fan-out and be enforced at disclosure

**Evidence: OBSERVED** — inspected worker allocation and passing mocked-worker/provider tests.

Possibly divides model/tool call allowances among candidates and gives them a shared deadline. Its mocked-worker test confirms total allocated calls stay within the batch grant and a failed sibling does not discard a successful candidate. Outtake separately gates disclosure of frames and source scope, and rejects another provider request before it exceeds its model-call or text allowance.

Evidence: Possibly [allocation](https://github.com/robotdad/possibly/blob/28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b/src/possibly/fanout.py#L102) and [test](https://github.com/robotdad/possibly/blob/28d82b9e00bf39cb9c3384b53cbaac0b1d441f2b/tests/test_fanout.py#L59); Outtake [disclosure and limit tests](https://github.com/robotdad/amplifier-smart-tool-outtake/blob/d441a37faea2a7fb762f2ca0395709ff1b542ab3/tests/test_intelligence.py#L50).

**Lesson:** declaring expected cost and enforcing an allowance are different capabilities. Child workers must not each receive the whole parent allowance. Permission to read local material also does not automatically permit sending it to a provider. These are call/byte/time limits, not evidence of a dollar-accurate cost ceiling; this review did not exercise real concurrent provider failures.

## 5. Portable output must survive loss of the creator's environment

**Evidence: OBSERVED** — fresh-store pack exchange and deterministic portable-export tests.

Unfold's pack test exports an asset, deletes the original file, imports into a fresh library, and verifies retained bytes. Reimport is idempotent; conflicting content is refused. A separate test checks explicit omission of assets without known redistribution rights. Stories produces identical ZIP hashes on repeated export and rejects fixtures containing hidden network dependencies.

Evidence: Unfold [round trip](https://github.com/robotdad/amplifier-smart-tool-unfold/blob/8fb14bc81e6d0f892963832c53aa598401b19370/tests/test_workflows.py#L17) and [omissions](https://github.com/robotdad/amplifier-smart-tool-unfold/blob/8fb14bc81e6d0f892963832c53aa598401b19370/tests/test_workflows.py#L120); Stories [portable export test](https://github.com/robotdad/amplifier-smart-tool-stories/blob/3b82a038c5aa321a126a6243fdb00b4168e0e7c2/tests/test_media.py#L193).

**Lesson:** the [output proposal](../proposals/output-is-bigger-than-the-response.md) could distinguish a preview, a reference to retained content, and a portable deliverable. A file path alone does not convey which one it is. These are same-tool round trips and export checks, not proof of a general interchange format or a demonstrated Unfold-to-Stories workflow.

## 6. Deterministic checks establish specific properties, not overall truth

**Evidence: OBSERVED** — evidence-delivery, source-classification and arithmetic tests; the scope recommendation below is judgment, not a comparative quality result.

Outtake rejects candidate evidence that was generated locally but never delivered to a completed model call. Even after delivery, the candidate remains `model_proposal` with `human_confirmed=false`. Stories recomputes supported arithmetic from evidence-linked inputs and prevents model extraction from upgrading a caller's summary into a primary source. Its human acceptance record remains separate from artifact checks.

Evidence: Outtake [test](https://github.com/robotdad/amplifier-smart-tool-outtake/blob/d441a37faea2a7fb762f2ca0395709ff1b542ab3/tests/test_intelligence.py#L99) and [submission validation](https://github.com/robotdad/amplifier-smart-tool-outtake/blob/d441a37faea2a7fb762f2ca0395709ff1b542ab3/src/outtake/intelligence.py#L199); Stories [classification and calculation tests](https://github.com/robotdad/amplifier-smart-tool-stories/blob/3b82a038c5aa321a126a6243fdb00b4168e0e7c2/tests/test_contract_gaps.py#L49).

**Lesson:** this contributes to [open question 3](../open-questions.md#3-does-the-deterministic-check-rule-generalise), but does not settle it. Correct arithmetic can use unsuitable inputs; delivered frames can miss the remembered event. We should report what was checked, against which evidence and revision, while keeping semantic uncertainty and human acceptance distinct. These implementations support a narrower claim than “a cheap deterministic check makes a capability safe unattended.”
