# Building an audio mastering smart tool

`aud` — [colombod/amplifier-smart-tools-audio](https://github.com/colombod/amplifier-smart-tools-audio),
MIT, 0.9.0, 330 tests, conformance 16/16.

Built in one day from nothing. Audio mastering and cleanup: measure a finished programme,
then run it through an ordered chain — detect and cut silences and filler words, de-ess,
de-verb, gate, expand, EQ, EQ-match to a reference, multiband compress, saturate, add
controlled ambience, hit a loudness target, hold a true-peak ceiling. Fifteen stages, one
render pass, one shell command.

What follows is what the spec did not tell us, and what the tool's own test suite could not
tell us either. The second category turned out to be the larger one.

---

## The licence wall is load-bearing for the whole "smart tool" idea

**Evidence: MEASURED** — three `deep-research` runs, `dr-d9ddb3ef`, `dr-6ec3b20e`,
`dr-299f0b34`, ~$0.60 total, reports on disk. N=3 independent queries, cross-checked against
each library's own PyPI/repository licence metadata.

Before writing a line we asked what a permissively-licensed audio tool can actually depend
on. The answer was uncomfortable and it generalises well beyond audio:

| library | what it does | licence |
|---|---|---|
| pedalboard (Spotify) | the obvious answer — EQ, compressor, limiter, reverb | **GPL-3.0** |
| matchering | reference EQ matching, i.e. one of our headline features | **GPL-3.0** |
| Rubber Band | the quality bar for time-stretch and pitch-shift | **GPL-2.0-or-later / commercial** |
| essentia | analysis toolkit | **AGPL-3.0** |

Every mature ready-made option in this domain is GPL-family. "Spotify open source" is not
"permissive", and a `pip install` that resolves silently tells you nothing about what you
just agreed to.

So the tool owns its DSP. The dependency list is numpy (BSD), scipy (BSD), soundfile (BSD-3),
pyloudnorm (MIT), and the Linkwitz-Riley crossovers, the multiband compressor, the
oversampled true-peak limiter, the phase vocoder, the FDN reverb, the de-esser, the
dereverb, the gate and the expander are written here.

**The finding for the spec:** a smart tool is defined as shipping its own capability, and the
spec is silent on what a tool may depend on to do that. In any domain where the mature
implementations are copyleft, "package a workflow as a tool" quietly means "reimplement the
domain". That is a real cost and it is invisible until someone reads the licences. It is
worth the generator or the docs asking the question out loud, because the natural first move
— `pip install` the obvious library — is the one that relicenses your tool.

The one permissive quality tier we did find, `python-stretch` (Signalsmith, MIT), is wired in
as an optional extra rather than a dependency. See below for what it cost.

---

## A contract between two agents is not a contract until something renders

**Evidence: MEASURED** — four separate parallel builds, each two or three agents against one
written contract. Seam defects per build: 3, 0, 3, 11. N=4 builds.

The tool was built in parallel lanes: one agent on the CLI and plan document, another on the
DSP engine, both coding against a written contract document. It works — but the contract
drifted every time, and never in a way that reading caught.

Round one, three defects: a `lib` capability with no CLI route (`analyze` simply did not
exist as a verb), a params-shape mismatch (`{"bands": [...]}` written, `params["crossovers"]`
read), and an unwrapped `ValueError` reaching stdout as a Python traceback in a tool whose
contract promises exactly one JSON document.

Round four was the instructive one. Reconciling the code against the contract document
turned up **eleven** divergences, including these three:

- a contract-conformant single-band plan (`"crossovers_hz": []`) **crashed at render**,
  because the splitter raises on an empty list;
- `limit.oversample` was documented, defaulted, and **silently never read** — a hand-written
  plan setting it got no effect and no warning;
- per-band defaults for omitted fields came from a dataclass, not the contract, so a
  hand-written plan got different processing than the document promised.

Every one of those had existed for days with a green suite, because the builder and the
engine agreed *with each other*. The contract is the published interface and nobody was
testing against it.

**What fixed it**, and it is one test: hand-write a plan document using **only the field
names, shapes and defaults the contract states** — never through the builders — and render
it. That asserts the property that was actually broken. Asserting key names in isolation
would not have caught the empty-crossover crash or the ignored `oversample`.

**The finding for the spec:** the conformance kit checks the tool's *outer* surface — manifest,
envelopes, exit codes, credential-free capabilities — all of which passed 16/16 throughout
every one of these defects. A tool that publishes an internal contract (our plan document, our
regions document) has a second interface the kit cannot see. The kit is right not to guess at
it, but the spec could say: if your tool publishes a document format, a round-trip test
written from the document alone is the check that matters.

---

## One shell command, not a conversation

**Evidence: MEASURED** — a 15-stage chain rendered in one pass; the editing chain verified
end to end on real speech in a container. N=1 per chain.

Borrowed from our video tool and confirmed to generalise. Every verb but `render` appends to
a plan document and passes it on; nothing touches a sample until `render`, which applies the
whole chain in a single pass:

```bash
aud detect silence in.wav \
  | aud cut --snap transient --pad-in 60 --crossfade 10 \
  | aud deess --amount 6 \
  | aud eq --hpf 60 \
  | aud compress --bands 150,1200,6000 --ratio 2.5 \
  | aud loudness --target -14 \
  | aud limit --ceiling -1.0 \
  | aud render in.wav out.wav
```

All fifteen stages ran in one pass and landed at −14.00 LUFS, −2.81 dBTP against a −1.0
ceiling, each stage reporting its own measurements.

The reason this matters for an agent caller is arithmetic: the alternative is a round trip
per stage — latency, tokens, and another chance to lose the thread — plus an extra decode and
encode each time, which is another quantisation and another chance to clip.

Three mechanisms serve the one idea, and they are for three different callers:

| mechanism | for a caller that |
|---|---|
| the pipe chain | knows what it wants |
| `aud preset show podcast \| aud render in.wav out.wav` | wants a known-good chain |
| `aud master in.wav out.wav --target -14` | wants the decision made for it |

**The finding for the spec:** "one command" is not a CLI style preference, it is what makes a
tool cheap for an agent to call. The spec's examples are mostly single-verb. A worked example
of a *chainable* tool — where the intermediate document is the contract and one terminal verb
does the work — would be worth more than another single-verb sample, because the design
decision (what travels between verbs) is the hard part.

---

## The model never sees the audio

**Evidence: MEASURED** — live Anthropic calls, both directions. N=2 files × 2 model tiers.

`aud advise` runs the deterministic analysis, hands the **measurements** to a model, and gets
back a plan with a stated reason per stage. It never touches a sample. Because it emits a plan
document, it pipes straight into `render`, so the decision is inspectable before anything is
written.

Real output, hissy material:

> `expand`: noise_floor_dbfs −30.5 vs integrated_lufs −13.06 is only ~17.4 dB separation
> (well under the 25 dB clean threshold), so the quiet passages need gentle downward
> expansion before loudness/limiting amplify that floor.

And on clean material (52.8 dB separation) it chose no gate at all. **The negative case is the
one worth reporting** — a tool that always reaches for a gate is worse than one that has none,
because gating clean material is damage.

Two things this shape buys: the DSP stays deterministic and testable, and the expensive part
is the only part that needs a credential. Every other verb runs with none.

**Model tier changed the answer.** Given identical prompts and identical measurements, the
cheap default did not reach for the gate on the hissy file; the larger model did, and cited
the right numbers. The prompt was not the problem. A smart tool's *defaults* therefore encode
a quality/cost decision that its manifest does not currently have anywhere to state.

---

## A stale default model is invisible to every test you will write

**Evidence: MEASURED** — first live call after shipping, HTTP 404. N=1, and one was enough.

0.6.0 shipped with a CHANGELOG line saying no live provider call had ever been made. We made
one. It returned:

```
HTTP 404 {"type":"not_found_error","message":"model: claude-3-5-haiku-20241022"}
```

against a perfectly valid key. The default had been chosen by reading documentation, never by
calling. Worse, the error told the caller to check their credential — sending them to debug
the one thing that was working.

The second defect, found minutes later with a reasoning model: our provider client read
`content[0]["text"]`. `content` is a list of **blocks**, and a thinking model returns a
`thinking` block first — so an entire tier of current models came back as "response did not
have the expected shape". Every test passed throughout, because every test used a fake that
returned a single text block.

**The finding for the spec:** the spec is careful that deterministic capabilities must run
with no credential, and the kit checks it. It says nothing about the model-backed ones, which
carry two things that rot: a **model name** and an **API response shape**. Neither is
checkable by any test that does not make a real call. A conformance check cannot make that
call — but a tool could be required to *declare* its default model somewhere machine-readable,
so staleness is at least greppable rather than discovered by a user.

---

## Never hand-write a mock. Replay a recording.

**Evidence: MEASURED** — three defect classes, each shipped past a green suite; then a
recording pass that produced five more findings on first contact. N=3 escapes, 18 recordings.

This is the finding I would keep if I could keep only one, and it was the hackathon steward's
ruling, not our discovery — we just paid for it three times in one day with 284 tests passing
throughout.

| what was faked | what the fake could not produce | what it cost |
|---|---|---|
| faster-whisper transcript | a word with `start == end`; any sample rate at all | one zero-duration word destroyed the **entire** regions document (3 runs in 5 on a 58 s file); and every timestamp was up to **3× out** on any file that was not 16 kHz — feeding straight into `cut`, so the tool cut the wrong part |
| an LLM provider response | a `thinking` content block | every reasoning model unusable in production |
| Signalsmith `timeFactor` | our own belief being the reciprocal of the truth | the stub **confirmed** the bug instead of catching it: `--factor 1.2` shortened audio |

The shape is always the same. **A fake built from the fields your own reader consults can
only confirm the shape you already assumed.** If the fixture was written by the same person
who wrote the reader, it tests the reader against itself.

The fix: record the real thing once, in a throwaway container, and replay it. `tests/replay.py`
is one shared harness over 6.2 MB of captures — 18 real faster-whisper transcriptions at four
sample rates, the real Signalsmith length table plus a real audio pair, four real provider
envelopes — each carrying provenance: library version, source audio sha256, capture date, and
the command that produced it. The harness **fails loudly** on a missing recording; the silent
fallback to a made-up value is the exact defect being removed. Result: 330 tests, **zero
skips** (three Signalsmith tests had been skipping because the library is not installed on the
dev box — the recording is, so they replay).

Things only the real data told us, now encoded as tests: **silence returns one hallucinated
word**, not nothing, while a 1 kHz tone returns zero segments. `timeFactor` reads back as
float32 (`0.8` → `0.800000011920929`). `inputLatency`/`blockSamples` are **methods, not
attributes**. Nobody writes a fake that surprises them.

Two traps worth passing on:

1. **A replay is bound to its exact input bytes.** The same speech at 16 kHz and at
   48-kHz-resampled-to-16 kHz produced *different transcripts* — 168 words vs 106, one
   degenerate word vs none. Same content, same model; only the resampler's output bits
   differ. A replay asserting recorded output against *similar* audio tests nothing, which is
   why every recording carries its source's sha256. Confirmed independently: regenerating a
   source wav from the documented `ffmpeg` command produced a different sha256, because the
   ffmpeg build differs.
2. **A replay tidier than the real library is a fake again.** Reproduce the awkward parts on
   purpose.

**The finding for the spec:** a smart tool wraps other people's libraries — that is close to
the definition. The conformance kit necessarily checks the tool's own surface, so the seam
where a tool meets its dependency is exactly where nothing checks and exactly where our real
defects lived. Recording-and-replay is cheap, and the recordings double as documentation of
what the dependency actually does. Worth a paved path.

---

## An assertion that cannot fail

**Evidence: MEASURED** — the same chain on two hosts, 0.97 dB apart, every assertion green on
both. N=2 hosts.

Running the identical chain on the identical commit:

| | dev box | clean container |
|---|---|---|
| integrated LUFS | −14.01 | −14.0153 |
| true peak dBTP | **−2.04** | **−3.0063** |

Both reported `ceiling_ok: true`. Every assertion passed at either value, and would have
passed at −1.2, and at −5.0.

The reason is structural and worth keeping. `loudness` is **closed loop**: it measures,
computes a gain, and lands on its target regardless of what happened upstream — which is why
LUFS matched. `limit` only **clamps**; when the signal never reaches the ceiling it does
nothing, so the reported true peak is an uncorrected byproduct of whatever the compressor and
saturator did, and different BLAS builds move it. A closed-loop stage hides host variation; an
open-loop one publishes it.

Six test sites had the same shape. The rule now enforced: **a chain whose limiter never
engages is not evidence that limiting works.** Each such test carries an engagement canary —
if the measured gain reduction is zero, the test fails saying the signal did not exercise the
limiter, and to fix the fixture rather than the assertion.

A seventh turned up in the audit: `verify()` used a zero-tolerance ceiling comparison while
the limiter used a documented 0.05 dB one, so a limiter working exactly as designed could be
reported as failing. Two independently written checks on one quantity, disagreeing. One
constant now, imported by both.

**The finding, general:** a boolean standing in for a measurement is the same defect as a
hand-written mock, one altitude up. Neither can surprise you.

---

## Smaller things, recorded so nobody pays for them twice

**Evidence: OBSERVED** unless marked.

- **An unbuilt verb must answer about the capability, not about its flags.** Three verbs were
  registered with no arguments at all, so `aud advise in.wav` — the invocation their own
  `--help` printed as the example — returned `usage_error: unrecognized arguments`. Bare
  `aud advise` answered correctly, which is why nothing caught it. The sweep test that fixes
  this class runs **every verb at the example its own documentation prints**, driven off the
  registered-verb list so new verbs are covered without editing the test.
- **A planned gap reported as a bug in the tool.** Rendering an unimplemented stage raised a
  bare `ValueError`, which the catch-all wrapped as `internal_error` with the remedy "this is
  an internal bug… report it" — while the message itself said the stage was planned for a
  later release.
- **A missing input file was reported as our internal error** across six entry points,
  telling the caller "not something wrong with your input" when the input was exactly what
  was wrong.
- **MEASURED — the install story is worth proving in a container, not assuming.** `aud` pins
  scipy≥1.14 (needs numpy≥2) while faster-whisper pulls onnxruntime, which historically pinned
  numpy<2. Four install shapes, machine-compared package diffs: **numpy and scipy did not move
  by a single patch version** under any extra. The exit code was never the evidence; the diff
  was. N=4 install shapes.
- **MEASURED — a click test built on a round number tests nothing.** Cutting exactly 1.000 s
  of a 440 Hz tone is exactly 440 cycles, so the phase matches across the join by construction
  and an unaligned splice scores the same as an aligned one. At 440.5 cycles the two sides meet
  in antiphase: `snap none` gives a sample step of 0.998875, `snap zero_crossing` gives
  0.031340 — the tone's own baseline. 31.9× apart. N=1.
- **Installing an optional extra silently changed behaviour.** The Signalsmith engine is
  selected automatically when present, so installing an optional *quality* tier reversed the
  meaning of `--factor`. An optional dependency that changes results, not just quality, needs
  saying out loud.

---

## What this tool still cannot claim

Recorded here because the rest of this document argues for exactly this.

- Only **Anthropic** has been exercised live among four provider backends. The other three
  are reviewed and uncalled — and the content-block defect above is precisely the kind of
  thing waiting in them.
- The four provider recordings have a documented manual recapture, not a script.
- Two synthetic values remain in the suite, both testing **our** code rather than a library's:
  a negative-duration word (no real recording produced one — only zero-duration), and a
  backend returning deliberately invalid text, which no successful recording can supply
  because recordings only capture calls that worked.
- Transient detection produces spurious onsets on a pure sustained tone. Adequate for edit
  placement; not a publication-grade onset detector.
- `dereverb` moves a synthetic 400 ms room from −27.9 dB/s toward the dry reference at
  −43.2 dB/s. Reducing a moderate room is achievable; removing a heavy one without artefacts
  is not, and the docs say so.
