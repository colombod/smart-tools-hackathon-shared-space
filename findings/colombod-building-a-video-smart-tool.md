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

---

# Second round: what a machine without credentials found

Everything above was learned on the machine that built the tool. This section is what
changed when it was installed from its git URL onto a container that had never seen the
checkout — no `gh` binary, no `~/.config/gh`, no netrc, no provider variables, **nine
environment variables in total.**

## The "works without a provider" promise is only testable where no credential store exists

**Evidence: MEASURED** — full deterministic chain on a credential-free container. **N=1
container, one arm.** Upgrades the OBSERVED finding above, which could only report that our
local test was insufficient.

Earlier in this file we recorded that our own no-provider test passed for the wrong reason:
the scrubbed environment passes `HOME` through, and an SDK reading credentials from a store
under `HOME` finds them.

On a container with nothing, the chain ran clean — `stitch --transition dissolve | render |
verify` gave 5.24s, audio at -21.3 dB, and a measured blend. The promise holds. **But we
could not have known that here**, and neither can anyone else running the same kind of local
check.

**For the conformance kit:** `loads-without-provider` passes today by scrubbing environment
variables. A tool that reads credentials from a file under `HOME` passes that check while
still depending on a provider. The check cannot detect this from inside the same machine.
Either it needs to run somewhere without a credential store, or it needs to say what it
does *not* cover.

## A refusal message can point somewhere that does not contain the answer

**Evidence: MEASURED** — a real defect in our shipped tool, found on first contact with a
credential-free machine, fixed and re-verified in the same container.

Asked to pick a transition from a description with no provider configured, our tool
correctly refused and told the caller to run our `check` command to learn how to configure
one. **That command never mentioned a provider.** It inspected two things and returned.

The instruction was a dead end, and it had been shipped that way.

It is not findable on a developer machine, and the reason generalises: **every machine that
builds a tool has the credentials that make its refusal paths unreachable.** A refusal only
fires where the thing is missing, so its guidance is only testable there too.

**For the spec:** deterministic-path conformance is checked. Nothing checks that a tool's
*own remediation instructions* lead anywhere. A smart tool is supposed to be consumable by
an agent that cannot ask a human what to do next — which makes a dead pointer more expensive
here than in ordinary software, because there is nobody to work around it.

## Over-stating a credential requirement is worse than under-stating a feature

**Evidence: OBSERVED** — our manifest marked a capability model-backed; on the
credential-free container it served a real query and labelled its own answer `[literal]`.

We listed `find` as model-backed. On the container it answered with zero credentials,
through a literal-search tier, and said so in its own output.

That is a false claim in the direction that matters most. A consumer reading "model-backed"
concludes they need a provider to search a video. They do not — for the common case, because
people searching a recording remember **words**, not paraphrases.

**For the spec:** `model_backed` is a boolean, and real capabilities are **tiered**. Ours
has a deterministic path that handles most queries and escalates only when the caller's
words are not the speaker's. There is no way to say that, so you must choose between two
wrong answers. We picked the wrong one, and a stranger's machine is what told us.

## Install success and install identity are different claims

**Evidence: MEASURED** — resolved commit compared across three independent sources. **N=1.**

`uv tool install git+<url>` exiting zero says the install worked. It does not say it
installed *what you pushed* — a cached wheel or a fast path could resolve elsewhere.

The check that does say it: the SHA `uv` reports building, the `direct_url.json` in the
installed dist-info, and `git ls-remote HEAD` captured **before** launch. All three matched.

**For the kit:** worth considering as a conformance check in its own right — a tool
installed from a ref should be able to prove *which* ref.

## The kit runs one more check against an installed tool than against a checkout

**Evidence: MEASURED** — same tool, same spec commit: **15** checks locally, **16** in the
container. N=1 each.

Locally we get 15 pass / 0 fail / 0 skip. Installed on the container, 16 pass / 0 fail / 0
skip. One check simply does not exist locally, and nothing in the output says so.

We have not yet identified which check it is, and we are recording the discrepancy rather
than guessing. **The point stands regardless:** a green local run and a green container run
are not the same evidence, and the summary line gives a reader no way to tell them apart.

---

# Third round: what a bigger surface taught us

The tool grew from eleven verbs to twenty-one — narration, vision indexing, colour
transfer, look grading. That growth is what surfaced these; none of them were visible when
the tool did one thing.

## The pattern that makes a model-backed capability shippable

**Evidence: MEASURED** — three independent instances in one tool, each with the figure that
decided it. **N=3 mechanisms, one arm each.**

We now have three capabilities where a model produces something unbounded, and all three
ship. What they share is not prompt quality. It is that **the tool can check the output
with arithmetic**:

| capability | what the model produces | what the tool measures | the number |
|---|---|---|---|
| generated transition | an ffmpeg expression | frame sampled mid-blend, distance to each side | 5 of 6 correct; the 6th refused |
| narration | a line per timed slot | spoken duration against the slot's budget | 4.25s into 6.00s |
| colour transfer | *(no model at all)* | statistical distance to the reference | 98.4% of the gap closed |

The third row is the control, and it is the interesting one: it needed no model, and we only
noticed *because* we had built the measurement first and could see the model was not
carrying any weight.

**The rule this suggests for the spec:** a model-backed capability is safe to ship
unattended when there is a **cheap deterministic check on its output**. Where that check
exists, generation becomes defensible. Where it does not, it is a coin flip with good
manners — and the manifest currently gives a consumer no way to tell those apart.

## The intelligence seam was more capable than its schema

**Evidence: OBSERVED** — a probe run before building on it; the described frame came back
verbatim correct.

We needed vision. `AgentRequest` has **no image field** — prompt, model, workspace,
output_schema, timeouts. The obvious conclusion is that the contract does not support
vision and needs extending.

It does support it. The request carries a **`workspace`**, and the agent behind it has
file-reading tools. Write frames to a directory, point the agent at it, and vision works. We
probed it first with a frame carrying the text `INVOICE 4471` and got exactly that back.

**Worth writing down because the wrong conclusion is the natural one.** A builder reading
that schema would reasonably decide the seam cannot do vision and either fork it, add a
field, or reach for a provider SDK directly — all of which break the boundary the scaffold
exists to create.

**For the spec:** when a contract's capability exceeds its field list, say so. One sentence
in the interface docstring — *"a workspace makes any file readable, including images"* —
would have saved the investigation, and would stop someone else adding a field that is not
needed.

## `model_backed` is a boolean; real capabilities are tiered *and* plural

**Evidence: OBSERVED** — one tool, four distinct provider capabilities, none expressible.

Round two noted `model_backed` cannot express a tiered capability. The bigger surface makes
the second half of the problem obvious too — this one tool now uses **four different kinds
of intelligence**:

| capability | kind | where it runs | credential |
|---|---|---|---|
| transcription | speech-to-text | **locally** | none |
| narration voice | text-to-speech | **locally** | none |
| choosing / writing / narrating | text reasoning | remote | provider |
| describing frames | vision | remote | provider |

A consumer asking *"what AI does this tool use?"* gets a four-row table, and two of those
rows need no credential at all. The ROADMAP question — *how do we make others aware of what
AI providers a smart tool uses* — has no single-sentence answer once a tool does more than
one thing, and `requires[]` cannot say which capability each entry unlocks.

**Concretely:** our manifest lists four `requires[]` entries, and the only place the mapping
from entry to capability exists is prose we wrote by hand in the `purpose` field.

## Friction: local inference is viable, and the method matters more than the choice

**Evidence: MEASURED** — clean-container verification of two local engines coexisting.
**N=1 container.**

Both local tiers work, with no credentials, and the numbers are comfortable:

```
faster-whisper   transcription   ~23x real time, 392 MB venv
piper-tts        synthesis       ~17x real time, +46 MB, 60 MB voice
```

They share an `onnxruntime` without a fight — **zero version changes** when the second was
installed, confirmed by a machine-readable package diff, then by running both engines in one
process.

**The method is the finding, not the packages.** We verified the install in a container
*before* designing around it, as a separately-tracked piece of work. Had it conflicted, the
whole narration design would have needed a cloud TTS and lost its credential-free promise —
and we would have found out after building it.

**And a wart any local-ONNX smart tool will hit:** under a restricted cpuset — Docker with
`--cpuset`, most CI — onnxruntime prints a red `[E:...] pthread_setaffinity_np failed` line
at *every* session init. Exit 0, output correct, purely cosmetic. But it is an ERROR line,
and a caller watching stderr reasonably reads it as failure. The library offers no hook to
suppress it. We filter that one known line and pass everything else through, because
swallowing a real error to hide a cosmetic one is a far worse trade.

## Measuring found four defects that reading never would have

**Evidence: MEASURED** — four defects, each with the figure that exposed it.

Not a spec finding. A finding about **how to test a smart tool**, and the pattern was
consistent enough to be worth stating.

| what looked fine | what measuring said |
|---|---|
| shot detection threshold `0.4` | real cuts score as low as **0.076**; a 3-shot video reported **1 shot** |
| `--look warm` colour weights | Lab b* shift of **+0.23** — correct direction, invisible in practice |
| a "no provider" test passing | the environment leaked a credential store; nothing was being tested |
| every render, on every video | any file **without an audio track** failed — `-map 0:a` on a stream that does not exist |

The last one is the sharpest. Screen recordings routinely have no audio and are the
commonest thing this tool gets pointed at, yet **every single verb was broken for them** and
no test caught it — because every fixture we had built happened to have sound.

**And one where the measurement itself lied.** Grading a flat neutral grey reported two
*opposite* looks as byte-identical. Not a filter bug: `colorbalance` weights shadows,
midtones and highlights separately, and a flat frame has one tone for them to act on.
Running the two filter strings through ffmpeg directly — bypassing our tool entirely — is
what separated *"my tool is broken"* from *"my fixture is degenerate"*.

**For anyone building one of these:** the deterministic tiers are exactly the part a
conformance kit cannot check for you. It verifies a manifest and a smoke test. Whether your
verb does the thing it claims is yours to measure, and the measurement wants a fixture
chosen to make failure visible — which is not the same as a fixture that runs.
