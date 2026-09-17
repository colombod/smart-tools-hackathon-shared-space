# What a video smart tool taught us about the spec

Building `vid` — a video editing/curation smart tool — after `deep-research` and
`fact-check`. Different enough from those two to press on parts of the spec they never
touched: three AI capabilities instead of one, a hard system dependency, and a genuine
question about whether generated code can be trusted.

Written per this repo's rules: every `##` opens with its evidence tier, and MEASURED
carries its sample size.

---

## `requires[]` is a flat list, but alternatives are not the same *kind* of thing

**Evidence: OBSERVED** — hit directly while writing this tool's install story; four real
alternatives, none expressible today.

One capability — transcribe speech — has four ways to satisfy it, and they are not
variations of one thing:

| how | what it actually is |
|---|---|
| `faster-whisper` | a **Python extra**, arriving with the tool |
| whisper.cpp | a **binary on PATH**, installed by brew or a cmake build |
| a cloud API | an **environment variable** and an account |
| whisperX | a heavier Python extra, ~600 MB |

`requires[]` can name a package. It cannot say *"any one of these four"*, and it cannot say
*what each one unlocks*. A caller asking "what do I need for `find` to work?" has three
correct answers with three different shapes, and there is nowhere to put them.

Our earlier two tools each had one backend and never exposed this. It only appears when a
capability has genuine alternatives — which, for anything touching local inference, is
normal rather than exotic.

**What would help:** let a `requires[]` entry carry *alternatives* and *what it unlocks*,
and let the kind be something other than "package".

## The manifest cannot say *which* AI is used *for what*

**Evidence: OBSERVED** — three distinct provider capabilities in one tool.

This tool needs speech-to-text, vision, and text reasoning. They are different models, with
different costs, and a caller may have credentials for one and not another. The ROADMAP asks
"how do we make others aware of what AI providers a smart tool uses?" — and with one
capability the answer is a sentence. With three it needs structure, because the honest
answer is *"speech locally, reasoning via your provider, vision not configured"*, and that
is a table, not a sentence.

## A closed set is what makes a model-backed capability safe

**Evidence: JUDGMENT** — argued from three cases in one tool; the unbounded arm was
deliberately not shipped, so this is reasoning, not a measurement.

The useful line turned out not to be "use AI carefully". It is:

> **A model may CHOOSE from things that exist. It may not INVENT one.**

Choosing is safe because a wrong choice is *detectable*. Asked which of 58 transitions suits
"soft and dreamy", a model returns a name checkable against the list. Asked which transcript
passage discusses pricing, it returns an id we already timed — the timestamp was never its
to produce.

Inventing has no floor. A model asked for a timestamp returns a plausible number
indistinguishable from a correct one until somebody watches the video.

This produced a three-tier pattern that is not video-specific:

```
tier 1   caller names a thing from a closed set     deterministic, no provider
tier 2   caller describes it, model PICKS from set  bounded — wrong answers detectable
tier 3   model GENERATES something new              unbounded — needs verification
```

**For the spec:** a manifest says whether a capability is model-backed. It does not say
whether that model's output is *bounded*. Those are very different risks, and a consumer
deciding whether to run something unattended cares about the second one.

## Generated code can be trusted when arithmetic grades it — and the failure proves it

**Evidence: MEASURED** — 6 plain-language descriptions → 6 generated ffmpeg expressions,
each rendered and graded by property checks. **N=6, one arm.** Script and writeup in the
tool repo under `docs/experiments/`.

| outcome | count |
|---|---|
| correct motion | 5 |
| valid syntax, wrong motion | 1 |
| would not parse | 0 |

Nothing failed to parse. The category we most feared did not occur.

**The single failure is the finding.** "Fade to black, then up into the second clip"
produced an expression that reads as obviously correct — fade one input to zero, bring the
other up from zero; zero is black. It is not. The filter evaluates **per YUV plane**, so
driving every plane to zero gives black luma *and extreme chroma*. Sampling frames, the
"fade to black" never darkened.

That expression was well-formed, plausible, and would survive review by anyone who did not
happen to know that detail. A human reviewer waves it through; the renderer executes it
without complaint. **No model graded it.** Three sampled frames and a distance comparison
did.

**For the spec:** if smart tools are going to generate code, the interesting question is not
whether a model can write it — it mostly can — but whether the *tool* can check it without
a human. Where a cheap deterministic check exists, generation becomes defensible. Where it
does not, it is a coin flip with good manners.

## Our own "works without a provider" test passed for the wrong reason

**Evidence: OBSERVED** — a test failed, and investigating the failure showed the test had
been proving nothing.

The spec's central promise is that deterministic paths run with no provider configured. We
test it by running each verb in a subprocess with provider environment variables scrubbed.

A test asserting that a model-backed path *refuses* when no provider exists **failed** — the
path succeeded. The scrubbed environment passes `HOME` through, and an SDK that reads
credentials from a store under `HOME` finds them. The provider was never absent.

The failure was luckier than a pass. Had that path been broken for any unrelated reason, the
test would have gone green while proving nothing about degradation — occupying the slot
where the real question goes.

**For the conformance kit:** "runs with no provider configured" is checked today by
absence of environment variables. That is necessary and not sufficient, because credential
stores under `HOME` are reachable anyway. A tool can pass the check and still be quietly
depending on a provider.

## The kit installs nothing, so `cli_argv` must already resolve

**Evidence: MEASURED** — same distribution root, 15 pass / 0 fail / **5 skip** before, 15 /
0 / 0 after adding the venv's `bin` to `PATH`. **N=1.**

Five checks SKIPPED because the tool's entry point was not on `PATH`. Nothing failed, the
verdict was still PASS, and five checks silently did not run.

This is the second time this has bitten us across two repos. A SKIP that looks like a PASS
is exactly the failure mode our tools exist to prevent.

**For the kit:** consider making the summary line state skips as prominently as failures, or
reporting `PASS (5 checks did not run)` rather than `PASS`.

## A generator's real output is the decisions it makes unavoidable

**Evidence: OBSERVED** — scaffolded with the smart-tool creator, then compared against what
we had built by hand for two earlier tools.

The scaffold ships `intelligence/interface.py` as a Protocol, with a docstring saying the
library depends on that contract and never on an SDK. We reached the same split
independently in the research tools, from the opposite direction, and it took three
iterations.

Independent convergence on a boundary is the strongest evidence it is real. The scaffold's
value is not the files — it is that the boundary is the default shape rather than something
you discover.

**Counter-observation, and it cost real time:** the scaffold cannot protect you from
*guessing* at its contracts. Two call sites here built the intelligence request from memory
with the wrong field name; both would have failed on first real invocation, and neither
would have surfaced until a user configured a provider. A generated `ask()` helper — one
function that constructs the request correctly — would have prevented it.

## Where a model runs matters more than whether it runs

**Evidence: JUDGMENT** — a design invariant this tool adopted; the alternative was not
built, so this is an argument.

A model runs when the *plan* is built, never when the video is *rendered*, and the plan
records what was asked alongside what was chosen. Three properties follow:

- the same plan renders identically on a machine with **no credentials at all**
- re-rendering **never re-invokes a model** — no cost drift, no nondeterminism
- what the model chose, and why, is readable **before** the expensive step

**For the spec:** "is this capability model-backed?" is a weaker question than "*when* does
the model run, and is its output frozen into an artifact a human can read?" The second
determines whether a result is reproducible and reviewable. The manifest asks the first.

## Open question we could not answer: measuring on the thing that matters

**Evidence: OBSERVED** — our own test fixtures, compared against one real recording.

Synthetic fixtures are reproducible, committable, and CI-friendly. They are also *easier*
than reality: cleanly-articulated synthetic speech transcribes better than two people
interrupting each other.

We checked against one real recording locally and the tool held up. But that recording is
private and cannot ship, so CI runs on the easy case forever.

**This is not a video problem.** Any smart tool whose input is messy real-world content has
it: the fixture you can commit is the one that flatters you. We have no good answer, and we
are recording the gap rather than pretending the synthetic result covers it.
