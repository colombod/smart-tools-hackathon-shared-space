# Guidance and evaluation are different artifacts: what a dense domain does to a smart tool

**Who:** colombod
**What this came from:** a controlled expertise probe of
[`aud` 0.11.1](https://github.com/colombod/amplifier-smart-tools-audio) — a shipped,
catalogued, 16/16-conformant audio mastering smart tool — plus the guidance half designed
against the result. Full report and 40+ raw artifacts under `aud-expertise/` in the working
workspace; three commissioned `deep-research` runs cited by id below.
**Status:** the measurement half is done and rendered. The guidance half is **designed and
barely started — 3 files, no wiring, never mounted, never measured.** Every claim that the
guidance recovers the measured failure is labelled PROPOSAL and should be read as one.

In a **dense domain** — one where taste, decision and jargon are instrumental, like audio
production or 3D visualization — a smart tool alone strands its own intelligence, and a
guidance bundle alone cannot be regression-tested. The two halves belong in **different
artifact types**, for reasons that are structural rather than stylistic. The split is the
finding; the measurement below is what forced it.

---

## A conformant tool that never once made the move its domain exists for

**Evidence: MEASURED** — `aud advise` run as shipped against eight induced spectral defects,
3 trials each. N=1 tool, 8 defect variants + 1 control, 27 runs per model tier, 2 tiers.
Raw plans, stderr and grades in `aud-expertise/raw/armA_{default,sonnet}_records.json`.

Fixtures are pink noise, 10 s, 48 kHz, stereo, each variant re-normalised to −20.0 LUFS
**after** filtering so loudness is never the variable under test. Every induced defect was
measured back out of the rendered file before being used:

| variant | induced | measured Δ vs control |
|---|---|---|
| boxy | +6 dB bell @ 400 Hz, Q 1 | **+4.83 dB @ 403 Hz** |
| dull | −9 dB high shelf @ 8 kHz | **−6.76 dB @ 12 kHz, −7.47 dB @ 20 kHz** |
| essy | +8 dB bell @ 6500 Hz, Q 2 | **+5.52 dB @ 6451 Hz** |
| thin | −6 dB bell @ 180 Hz, Q 1 | **−5.08 dB @ 180 Hz** |

(four more: muddy, rumbly, honky, harsh — all verified, `raw/fixture_verification.json`.)

The defects are plainly present in the report the model actually receives
(`raw/analyze_all.txt`) — `boxy` shows 500 Hz sitting 3.4 dB above 1 kHz against a control
flat to within 0.4 dB across all ten octave bands.

Grading is on the **plan's stage parameters** — frequency, direction, gain — never on its
prose.

| arm | what changed | score |
|---|---|---|
| **A** — `aud advise` as shipped, default model (`claude-haiku-4-5`) | nothing | **0/24** |
| **A** — same, `--model claude-sonnet-4-5` | model tier only | **2/24** |
| **B1** — same model, the jargon word only, **no measurements at all** | measurements removed | **21/24** |
| **B2** — `aud`'s own prompt + **one sentence of intent** | one line added | **24/24** |
| **C1** — a neutral nudge, no jargon, **no API change** | one line added | **15/20** (4 refusals) |

Arm A's 0/24 is not a near-miss. **26 of 27 runs, control included, returned the
byte-identical two-stage chain `loudness + limit`**; the 27th added `expand`. The plan is
not a function of the spectrum. It is a function of the loudness target. The only defect
the sonnet tier ever caught was `essy` — the one backed by a dedicated named scalar
(`sibilance_ratio`), rather than by a position in an array.

B2's added sentence is exactly: `The client says: "<jargon>". Address that complaint.` Same
system prompt, same measurements, same model, same `advisor.advise`, same stage builders,
same validation. 0 refusals; every proposed chain passed `aud`'s own builders unmodified.

**So the expertise is in the model, and the tool's interface cannot reach it.** Not a tier
problem — sonnet scored 2/24. Not a measurement problem — the defects are plainly in the
report. Not a validation problem — nothing was refused.

---

## Rendering it confirms the failure at the audio, not merely at the plan

**Evidence: MEASURED** — each trial-1 plan rendered through `aud.lib.render`, then measured.
N=7 bell defects by neighbour contrast, 1 shelf defect broadband-relative.
`raw/neighbour_contrast.json`, `raw/rendered_measurements.json`.

| variant | defect before | **Arm A removed** | Arm B2 removed |
|---|---|---|---|
| boxy @400 | +3.98 dB | **+0.01 dB** | +1.27 dB |
| muddy @250 | +3.99 | **−0.00 dB** | +1.23 dB |
| honky @900 | +4.22 | **−0.00 dB** | +0.27 dB |
| harsh @3500 | +3.91 | **+0.01 dB** | +0.57 dB |
| essy @6500 | +6.70 | **−0.01 dB** | +3.26 dB |
| dull @12k | −2.51 | **+0.38 dB** | +1.86 dB |
| thin @180 | −3.59 | **+0.00 dB** | +1.49 dB |

**−0.01 to +0.38 dB of a 4–6.7 dB defect. That is measurement noise.** The rendered output
a user receives is their input, 6 dB louder and peak-limited. This is step 3 of the audit
ladder from
[`colombod-auditing-smart-tools.md`](colombod-auditing-smart-tools.md) doing what it did to
`vid`: nothing above it on that ladder can see a wrong answer.

Worth stating against my own B2 result: B2 scored **24/24 on parameters** and removed as
little as **0.27 dB** at 900 Hz. Both numbers are true and they answer different questions.
Grading parameters measures whether the tool *aimed* correctly; only rendering shows whether
it *hit*. A gate that stops at the plan will certify a tool that aims well and does nothing.

---

## An intent parameter alone buys an obedient tool, not a diagnostic one

**Evidence: MEASURED** — control C2: the same one-sentence intent channel fed a complaint the
file demonstrably does not have. N=5 files × 3 trials = 15 runs. `raw/controls_records.json`.

| file (real defect) | told | followed the false complaint | also fixed the real one |
|---|---|---|---|
| boxy | "harsh" | 3/3 | 2/3 |
| muddy | "dull, no air" | 3/3 | 0/3 |
| harsh | "muddy" | 3/3 | 0/3 |
| dull | "rumbly, too much low end" | 3/3 | 0/3 |
| thin | "sibilant, essy" | 3/3 | 0/3 |
| **total** | | **15/15** | **2/15** |

**It never once pushed back.** On the `harsh` file told "muddy" it boosted 4 kHz by +2.5 dB,
3/3, while its own reasoning stated that 4 kHz was the loudest band in the file:

```
  - eq: ... 4000 Hz is already brightest at -30.9 dB.
        ... gentle dips at 250/500 Hz with a boost at 4 kHz will cut mud and add clarity
        per client complaint.
```

This is why the honest recommendation is not "ship a `--goal` flag". C1 — a neutral nudge
carrying no jargon and no diagnosis, *"compare each octave band to its neighbours"* — recovers
**15/20 with no API change at all.** A tool that acts on a caller's words without checking
them against its own measurements is worse than one that ignores them: it is confidently
wrong rather than merely inert.

---

## The residual defect a parameter cannot fix: absolute numbers with no derived contrast

**Evidence: MEASURED** — variant `thin` under C1. N=1 variant × 3 trials, 3/3 identical.

The report shows 125 Hz = −37.0 and 250 Hz = −37.2 against ~−33.5 elsewhere — i.e. **3.5–3.7 dB
below** their neighbours. The model wrote:

```
  - eq: 125 Hz and 250 Hz are ~3.5-3.9 dB above their neighbors; ...
        rolling off below 31.5 Hz removes subsonic mud and tames the low-mid bump.
```

The magnitude is right to a tenth of a dB. **The sign is inverted, 3/3** — so it cut the very
hole that makes the file thin. `analyze` hands the model ten *absolute* octave values and no
derived contrast; the prompt's rule that a reason must cite a specific number
(`prompts.py:70-72`) is satisfied perfectly by reasoning that is backwards.

> ★ **Insight:** requiring a cited number buys traceability, not correctness. Here the model
> cited −35.13 dB at 31.5 Hz — a true number naming the *quietest* band in the file — and
> high-passed it as "excess subsonic energy". A citation rule constrains the *form* of a
> justification; only a comparison rule constrains its *direction*.

---

## What the spec is silent on, that a real tool had to answer anyway

**Evidence: OBSERVED** — read from `aud`'s own source. N=1 tool, 4 call sites.

| where | what is there |
|---|---|
| `advise` signature | `src/aud/lib.py:1415-1422` — `(path, *, target_lufs, ceiling_dbtp, reference_path, model, backend)` |
| `master` signature | `src/aud/lib.py:1490-1500` — same, plus `out_path`/`dry_run` |
| CLI surface | `src/aud/cli.py:297-301` — `path`, `--target`, `--reference`, `--model` |
| the prompt itself | `src/aud/intelligence/prompts.py:89` — no free-text channel exists to carry a caller's words |

The spec says a smart tool ships its own AI capability, and that its deterministic paths run
with no provider configured. `aud` satisfies both, and everything else: manifest present,
model output validated through the same builders as hand-written plans, a bad proposal
refused rather than corrected. **16/16 on the conformance kit. 0/24 on its own domain.**

The spec says nothing about **where domain expertise lives**, and nothing about **how a caller
expresses intent**. Those are not oversights in `aud` — they are questions the spec has not
answered, which a real tool had to answer anyway, and answered badly by default. A second
structural gap found while reading: `lib.eq` accepts `shelves` (`lib.py:765-770`) while the
stage reference shown to the model omits them entirely (`prompts.py:30`), so for "dull" and
"rumbly" the correct instrument is withheld from the only component that has to choose it.
A capability the library has and the intelligence cannot name.

---

## Why evaluation belongs in a smart tool and guidance belongs in a bundle

**Evidence: JUDGMENT** — argued from the sections above. The counter-arm (one artifact doing
both) is not built, so this is reasoning, not a result. Where it rests on a measurement I say
which one.

The probe above is a *thing*, not an essay: generate a fixture with an induced defect of known
centre and gain, run the tool under test, grade the returned parameters against ground truth,
compare against a **null control** (does it invent a defect in a clean file?) and a
**wrong-answer control** (does it comply with a false complaint?). Four properties follow:

1. **Its core is deterministic.** Fixture synthesis, filtering, rendering, third-octave
   analysis, neighbour-contrast arithmetic and grading involve no model at all. It passes the
   spec's own test — straight paths run with no provider configured — by construction rather
   than by discipline.
2. **Portability is the whole point.** Anyone should be able to probe any smart tool's
   expertise, not only Amplifier users. A bundle cannot be installed by a Copilot CLI user;
   `uv tool install` can.
3. **It is the only half that produces a number.** 0/24 → 24/24 → 15/20 → 15/15 is what makes
   the argument land. Nothing in a knowledge base can generate those.
4. **It is a regression test.** Once the fixtures are fixed, re-running it after a prompt
   change is a cost of seconds.

The guidance half is structurally the opposite. What it needs — modes, staged context loading
so the tonal vocabulary costs tokens only inside the agent that consults it, selective
activation, adversarial critics run cold — are **host mechanisms with no CLI equivalent.**
You cannot express *"load the tonal vocabulary only when the operator is in the tone stage"*
as a library function. Flattening it into a tool means shipping the whole knowledge base into
every call's context and losing the delegation boundary that makes it affordable.

**The strongest case against the split:** two artifacts is a seam, and seams drift. The
guidance can teach a band the probe never measured, and nothing will catch it. I accept that
cost, because the alternative — one artifact — loses either the measurement or the mechanism,
and the measurement above is the only reason any of this is more than an opinion. The
mitigation is provenance per entry, below, not a merge.

**What the split is NOT:** it is not "put the prompt in a bundle". C1 measured that
three-quarters of `aud`'s specific gap closes with one sentence in `prompts.py` and no new
artifact of any kind. The cheapest fix comes first. The split is for the part that remains
after that sentence: the vocabulary, the instrument context, the ordering rules, the
corroboration discipline that C2 says an intent parameter must ship with.

---

## Prior art: the guidance half already exists, done well, by someone who had the same problem

**Evidence: OBSERVED** — read from the repository. N=1 bundle: 15 agents, 10 domain docs,
11 committed research runs.

[`colombod/amplifier-bundle-3d-developer`](https://github.com/colombod/amplifier-bundle-3d-developer)
reached this shape independently, for 3D visualization — the same kind of dense domain. What
it gets right, and what this finding is following rather than inventing:

- **A composable behavior, not a root bundle.** `behaviors/3d-core.yaml` layers with `--app`
  onto whatever bundle you already run.
- **Engine-agnostic domain lenses, platform knowledge in separate layers**, so a web consumer
  never pays for Unreal knowledge in its agent catalog.
- **Three critics run cold and independent** — `context_depth="none"`, the same artifact
  rather than a summary, no critic told what the others were asked — and **contractually
  required to emit `file:line` evidence anchors**, with fixed worst-wins aggregation and no
  softening of a FAIL.
- **A `docs/research/` → `context/domains/` distillation**: eleven `deep-research` runs, 736
  sources, run id recorded in each file, evidence tags (`[doc]` vs generalization) carried
  through per claim, and each report stating plainly where the evidence ran out.

The one thing it does not have — **and this is a genuine limitation of the form, not a
criticism of that bundle** — is a measurement behind its knowledge. Its claims are researched
and reviewed, and reviewing is a real quality bar. But **reviewed knowledge can only be
reviewed again. Measured knowledge can be regression-tested.** There is no `3d` equivalent of
"we induced a known defect, asked the bench, and it scored 0/24" — because there is no probe.
That is the gap the smart-tool half fills, and it is the reason the two halves are worth
having as two artifacts instead of one.

---

## Provenance that reports its own weakness, demonstrated

**Evidence: OBSERVED** — three `deep-research` runs commissioned against the three things the
audio build had honestly flagged as UNRESOLVED. N=3 runs, depth medium, each read from its own
brief at `~/.local/state/amplifier-research/runs/<id>/brief.md`.

The discipline being demonstrated: **research what you know you do not know, rather than
re-researching what you measured.** Each entry in a knowledge base then carries one of three
provenances — measured (a probe run), researched (a run id), or convention.

| run | question | what came back |
|---|---|---|
| `dr-42bbfca9` | where is "honky"? | **Genuinely contested, and the reason is instrument context.** Vocal/mix guidance centres ~700 Hz–1 kHz within ~500 Hz–1.5 kHz; one influential chart family separately labels piano "honky-tonk" at ~2.5 kHz. Not an error — a context dependence. |
| `dr-0cbb1a37` | is crest factor a proxy for "punch"? | **A rough diagnostic, not a valid standalone proxy.** PSR (local) generally outperforms PLR (global); academic models add onset time, transient-to-steady-state ratio, inter-band ratio and duration. |
| `dr-60331d81` | are "dull" and "lacking air" the same complaint? | **Mostly distinct but not unanimously** — dullness broad across ~250 Hz–12 kHz, air a low-gain shelf ~10–20 kHz; but the Maag EQ4's own AIR BAND reaches down to 2.5 kHz and at least one guide prescribes an air boost as the cure for dullness, collapsing the distinction. |

**All three returned `medium`, and each states its own reason for the cap** in its brief: most
sources are informal EQ cheat-sheets rather than textbooks (`dr-42bbfca9`); the correlation
statistics rest on a single unreplicated study (`dr-0cbb1a37`); much of the evidence is generic
frequency-chart content rather than named-engineer testimony (`dr-60331d81`).

That is worth a sentence on its own. A knowledge base entry that says *"medium, because this
traces to one chart family not cross-validated anywhere else in the evidence"* is one the next
person can push on. Provenance that reports its own weakness is the kind you can build on;
provenance that reports only a band is indistinguishable from a guess with a citation.

`dr-42bbfca9` also settles an open item from the measurement: B1's only miss was `honky`,
where the model answered 2–3 kHz against a fixture bell at 900 Hz. **That was scored a miss
and it should not be read as ignorance** — the research says both placements are attested, and
which one is right depends on the instrument the tool was never told about.

---

## The loop: what the two artifacts do to each other

**Evidence: PROPOSAL** — not implemented and not measured. The guidance half currently exists
as **3 files, 873 lines, zero commits, no behavior or bundle wiring, never mounted, never
run.** The claim that it recovers the 0/24 is an argument, nothing more.

The probe measures. The guidance teaches. The probe re-measures. Concretely, what the
measurement above says the guidance must carry — each line traceable to a section of this file:

- **Diagnose by comparison, never by an absolute number** — from the 3/3 sign inversion.
- **A deficit is not an excess** — same.
- **State whether the measurements corroborate the complaint before acting on it** — from
  C2's 15/15 compliance and the +2.5 dB boost into the real defect.
- **Instrument context decides a contested band** — from `dr-42bbfca9`.
- Every vocabulary row tagged `[measured]` (with the induced centre, gain and a confirmation
  threshold that can be re-run) or `[convention]` (defensible, reviewable, **not falsifiable
  here**).

The value of the pairing is that the first three are not opinions about mastering. They are
the three failure modes a measurement caught, written down where the next run can check
whether writing them down helped. That is a knowledge base with a regression test attached —
which is the thing neither artifact type provides alone.

---

## What I could not settle

- **The guidance half is not built, so the recovery claim is unmeasured.** Three files exist;
  nothing is wired, mounted or run. Until the probe is re-run against a session carrying that
  guidance, "the split works" is a design argument sitting next to a measurement that only
  establishes the *problem*. Do not quote it as a result.
- **N=1 tool, in one domain, on synthetic material.** Every fixture is pink noise, which has a
  flat contour by construction; real programme material has a sloped one, so a `[measured]`
  threshold may not transfer. And audio is a domain where "good" is unusually measurable —
  frequency bands are numbers. **Whether the split holds where "good" is less objectively
  measurable is untested**, and that is precisely where a dense domain is most likely to need
  it.
- **The probe's interface has had exactly one consumer.** Its generality — whether
  induce-a-defect / grade-parameters / null-control / wrong-answer-control is a shape that
  fits any smart tool, or only one that renders audio — is unproven. I believe it generalises;
  I have not shown it.
- **Which half should own the corroboration rule.** C2 says an intent parameter must ship with
  one. It could live in the tool's prompt (cheap, invisible to the caller) or in the guidance
  (visible, reviewable, skippable). I did not build either, so I have no basis to choose.
- **No cost accounting exists on any `aud` surface**, so this experiment's ~135 model calls and
  ~350 s of wall time cannot be converted to a spend figure. That is a gap in the tool worth
  naming for any model-backed verb, and it means I cannot tell you what the probe costs to run.

---

## Replication at 0.12.0: the gap survived two releases, and the model is not the problem

**Evidence: MEASURED** — the same induce-a-defect / grade-the-parameters probe re-run against
`aud` **0.12.0** (six defects, one clean control), plus a direct question to the same backend
outside the tool. N=1 trial per defect, N=1 direct question. Raw plans and per-stage reasoning
in `evidence/colombod/expertise-0.12.0/`.

The probe above measured 0.11.1. Two releases later — after the resample/downmix work and
three rounds of spec-adherence fixes — I re-ran a smaller version of it. **The gap is still
there.** Six defects were induced into pink-noise-plus-harmonics programme material, each
verified present by Welch PSD before anything was asked to diagnose it:

| file | induced | measured delta vs clean, in its own band |
|---|---|---|
| `boxy` | +6 dB @ 400 Hz | **+5.6 dB** in 300–550 Hz |
| `rumbly` | +9 dB shelf < 80 Hz | **+7.6 dB** in 20–80 Hz |
| `dull` | 4th-order LPF @ 5 kHz | **−28.7 dB** in 8–16 kHz |
| `harsh` | +6 dB @ 6 kHz | **+5.4 dB** in 5–7.5 kHz |
| `muddy` | +6 dB @ 250 Hz | **+5.6 dB** in 180–350 Hz |
| `thin` | −8 dB @ 200 Hz | **−7.7 dB** in 150–280 Hz |

Graded on the returned plan's **stage parameters**, never its prose:

| file | grade | every EQ move the plan makes |
|---|---|---|
| `boxy` | **PASS** | 500 Hz −4.0 dB |
| `muddy` | **PASS** | 250 Hz −4.5 dB; 125 Hz −1.5 dB |
| `rumbly` | MISS | 8000 Hz −4.5 dB |
| `dull` | MISS | *no EQ stage at all* |
| `harsh` | MISS | *no EQ stage at all* |
| `thin` | MISS | 8000 Hz −3.5 dB; 125 Hz −1.2 dB |

**2 / 6.** Better than 0.11.1's 0/24, and the two passes are genuinely good — `muddy` was
corrected at exactly the induced frequency, and `boxy`'s reasoning names the band correctly
("The 500 Hz band is +5.65 dB above the file's own median"). But `rumbly` cut 8 kHz for a
sub-80 Hz problem with no high-pass; `dull` and `harsh` emitted no EQ stage at all against a
28.7 dB HF deficit and a 5.4 dB peak; `thin` needed a boost at 200 Hz and cut 125 Hz instead.

### The new arm: ask the same model directly, and the gap moves

The earlier probe fed intent through `aud`'s own prompt. This one asks the model **outside the
tool entirely** — same backend, same model (`anthropic` / `claude-haiku-4-5-20251001`, as the
tool itself reports on stderr), one question, no measurements attached:

```
{"boxy":   {"band":"500Hz", "fix":"cut"},     {"rumbly": {"band":"80Hz",  "fix":"cut"},
 "dull":   {"band":"4kHz",  "fix":"boost"},    "harsh":  {"band":"3kHz",  "fix":"cut"},
 "muddy":  {"band":"250Hz", "fix":"cut"},      "thin":   {"band":"200Hz", "fix":"boost"}}
```

Six for six on direction, and on band for five of six (`harsh` answers 3 kHz — the classic
harshness region — where this fixture's defect was at 6 kHz). `rumbly` → 80 Hz cut, `thin` →
200 Hz boost, `muddy` → 250 Hz cut: exactly the corrections the tool failed to make.

**The expertise is in the model. The tool scores 2/6 on defects whose vocabulary that same
model maps correctly 6/6, because the tool never asks.** `advise`'s signature is still
`(path, *, target_lufs, ceiling_dbtp, reference_path, model, backend)` — there is no channel
for intent, so the only question ever put to the model is "here are measurements, pick a
chain". This is an **interface** finding, not a model-capability one, and it is now localised
by construction rather than argued.

### A robustness defect the control file found

The clean control — the one file with nothing wrong with it — **failed**:

```json
{"error": {"code": "bad_model_output",
  "message": "Model response was not valid JSON: Extra data: line 15 column 1 (char 497)",
  "remedy": "Retry advise/master, or build the chain by hand with the deterministic stage verbs."}}
```

1 of 7 runs, on well-formed input, from a model that emitted trailing content after its JSON.
The failure is loud and names a remedy, which is the spec behaving as intended. But a
model-backed verb with no retry around a known-flaky decode shape will fail this way for a
caller, and nothing in the conformance kit or the adherence reviewer can see a defect that
only appears once every seven runs.

### The methodological warning, which cost me the first result

**Evidence: OBSERVED** — my own first grader, N=1.

My first grader scored **6/6 MISS** and printed `all eq moves: none` for every file. That
result was entirely false. The grader looked for `peak` / `freq` / `gain` / `hpf`; the real
schema is `peaks[].freq_hz`, `peaks[].gain_db`, `hpf_hz`, `shelves[]`. It matched nothing,
and reported matching nothing as total failure — with exactly the same confidence as the
correct grader that later found 2/6.

This is the repository's own lesson one level up. We say a boolean is not a measurement and a
hand-written mock can only confirm what you assumed. A **grader** built from field names you
guessed has the same defect, and it is more dangerous than a bad test: a broken test usually
goes red and gets looked at, whereas a broken grader produces a plausible, publishable,
completely wrong number. The only thing that caught it was refusing to write down 0/6 without
first reading one raw plan.

**If you build an evaluation harness for a smart tool, assert that your parser found
something before you trust what it says it found.** A detector that matches nothing must fail
loudly, not score zero.
