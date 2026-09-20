# Auditing smart tools: what the checks catch, and what shipped past all of them

colombod. From putting two already-"finished" tools — `aud` and `vid` — through
`smart-tool-creator`'s two checks, and then through something neither check does: rendering
the output and measuring it.

Both tools were conformant, tested, released and in the catalog before any of this started.
`aud` was at 0.10.0 with 418 passing tests. `vid` was at 0.3.1 with 175. Both passed the
conformance kit 16/16. What follows is what was wrong anyway.

---

## The conformance kit passing tells you almost nothing about shape

**Evidence: MEASURED** — both checks run on two tools. N=2 tools, 1 run of the kit each,
1 first run of the reviewer each.

| tool | `check-conformance` | `check-spec-adherence` |
|---|---|---|
| `aud` 0.10.0 | **16 PASS / 0 FAIL / 0 SKIP** | **7 adhere / 10 deviate** |
| `vid` 0.3.1 | **16 PASS / 0 FAIL / 0 SKIP** | **7 adhere / 10 deviate** |

Identical scores on both axes, from two tools built weeks apart in different domains. The
kit is 16 deterministic rules about the manifest, the descriptor and the process contract:
does `--help` exit 0 with the provider environment scrubbed, does a bad verb exit non-zero,
does the manifest version match the package version. Every one of those is worth checking
and none of them is about whether the tool is *shaped* like a smart tool.

What the kit said nothing about, in `aud`, while reporting 16/16:

- Error envelopes were written to **stdout**. Every `{"error": ...}` went into the same
  stream as results — so a failure mid-chain contaminated the plan a caller was piping into
  the next verb. The kit checks that a bad invocation exits non-zero. It does not check
  *which stream* the reason arrived on.
- `aud --help` — the file an agent reads to decide how to drive the tool — carried no
  install command at all.
- The shipped agent skill carried a capability-to-prerequisite matrix duplicating runtime
  help, free to drift from it.
- Two capability documents stated their capability was unimplemented and would return
  `not_implemented`. It had rendered since 0.4.0, six releases earlier.
- `advise` analysed the input file before checking for a credential; `detect fillers`
  decoded the entire file before checking whether the extra it needs was installed. A caller
  with neither paid the full decode to be told they were never able to run.

**The honest framing:** the kit is a floor, and it is a good floor. Treating it as a verdict
is the error. Both tools had been sitting at "conformant: yes" for weeks.

---

## The model-backed reviewer is a sampling process, not an enumeration

**Evidence: MEASURED** — three consecutive runs of `check-spec-adherence` against `vid`,
each after the previous round's findings were closed. N=3 runs, 1 tool.

| run | verdict | what it named |
|---|---|---|
| 1 | 7 adhere / **10 deviate** | library/CLI split, capability skills, thin skill, manifest, artifact paths, stdout/stderr, remedies, partial results, prerequisites, temp files |
| 2 | 13 adhere / **4 deviate** | capability skills *again* (different verbs), artifact paths *again* (different capabilities), remedies *again* (different files), prerequisites *again* (different path) |
| 3 | 11 adhere / **6 deviate** | remedies *again* (`color.py`, untouched until now), partial results *again* (a genuine new bug), prerequisites *again*, artifact paths *again* (env-var-derived roots), plus a false statement in the rendered skill |

Read the middle column and it looks like regression — 4 deviations became 6. It is not. The
**classes** recur; the **instances** differ. Each run found real, previously-unnamed
occurrences of the same rules in files the previous run had not sampled. Run 2 fixed
`verify.py` and `narrate.py` for missing remedies; run 3 found the same defect in
`color.py`, which nothing had looked at yet.

One of run 3's findings was a real bug no earlier run mentioned: `index.py` ran the ffmpeg
scene-detection command and never checked `result.returncode`, so a failed detection
persisted an index claiming a single shot spanning the whole video — indistinguishable from
a genuine single-shot result.

**The consequence for anyone using this tool:** a single clean-ish run is not a pass. Run it
until a run names nothing new, and treat each finding as *"here is one instance of a class
that probably has siblings"* rather than a bug ticket. Every time we swept for siblings of a
named finding, we found some.

**Reading the output correctly matters too.** The reviewer's summary line prints only the
*first* evidence item, and for several checks that first item is a **positive** observation
with the actual defect in evidence lines 2-4. We nearly dismissed three findings that read
as compliments in the summary.

---

## Neither check touches whether the tool produces the right answer

**Evidence: MEASURED** — live renders against a generated 8.000 s / 640x360 / 25 fps clip,
durations from `ffprobe`. N=1 fixture, 9 verbs, before/after each.

This is the finding that matters most, and it cost the least to discover.

`vid zoom` **multiplied video duration by fifty**:

```
$ vid zoom a.mp4 --to 1.3 --at 0:02 --duration 2 | vid render z2.mp4
$ ffprobe -show_entries format=duration z2.mp4
400.000000          # from an 8.000000 s source
```

Three distinct defects in one filter:

1. ffmpeg's `zoompan` `d` is **output frames per input frame**, not the effect's total
   length. The code set `d = duration × fps` = 60, so every input frame fanned out into 60.
   The arithmetic is exact: 8 s × 25 fps = 200 frames, × 60 = 12000, at `fps=30` = 400 s.
2. `--at` was ignored entirely. No time gating; the ramp ran across the whole clip.
3. `zoompan` had no `s=`, so it silently defaulted to `hd720` and rescaled **640x360 to
   320x180**.

At the moment that shipped, `vid` had **175 passing tests**, a **16/16** conformance verdict,
and had been read end to end by a model-backed reviewer that produced ten accurate findings
about its shape. None of the three saw it.

They could not. The kit checks the process contract. The reviewer reads source and reasons
about structure. The test suite asserted the **plan document** and the **ffmpeg argv string**
— both of which were perfectly well-formed. `-filter_complex` containing `d=60` is valid
syntax expressing a wrong idea. The only thing that catches it is running ffmpeg and
measuring what comes out.

The sweep that found it takes about a minute to write:

```
trim         640 360 25/1 5.000000    expect 5.0
cut          640 360 25/1 6.000000    expect 6.0
zoom         640 360 25/1 8.000000    expect 8.0     <- was 400.000000
retime       640 360 25/1 4.000000    expect 4.0
vignette     640 360 25/1 8.000000    expect 8.0
recolor      640 360 25/1 8.000000    expect 8.0
stitch       640 360 25/1 13.000000   expect 13.0
audio-remove 640 360 25/1 8.000000    expect 8.0
audio-mix    640 360 25/1 8.000000    expect 8.0
```

**For the spec:** there is no conformance rule of the form "the tool's deterministic
capabilities produce correct output", and there probably cannot be a generic one — the kit
has no idea what `vid` or `aud` is *for*. But the spec could say that a tool's own suite
must exercise its capabilities **end to end against a real artifact**, not merely assert the
intermediate representation it hands to something else. Every defect in this document that
a test could have caught was in that gap.

---

## Isolated-verb testing misses composition, which is the advertised feature

**Evidence: MEASURED** — every combination of an audio-emitting verb followed by an
audio-dropping verb, rendered. N=1 fixture, 12 combinations.

`vid`'s headline claim is that verbs chain: *"every verb passes an edit plan, and one render
compiles it to a single ffmpeg pass."* That is the reason to use it over raw ffmpeg.

```
$ vid trim a.mp4 --from 0:01 --to 0:06 | vid audio remove | vid render comp.mp4
[fc#0] Filter 'asetpts:default' has output 1 (a2) unconnected
Error binding filtergraph inputs/outputs: Invalid argument
```

`trim` emitted its audio branch producing pad `[a2]`. `audio remove` correctly added `-an`
and dropped the audio map. The dead pad stayed in the graph with nothing consuming it, and
ffmpeg refuses to bind a graph with an unconnected output.

Sweeping the class rather than fixing the pair:

| combination | before |
|---|---|
| trim → audio remove | BROKEN |
| trim → audio replace | BROKEN + unbounded-audio hang |
| cut → audio remove | BROKEN |
| cut → audio replace | BROKEN + unbounded-audio hang |
| retime (speed) → audio remove | BROKEN |
| retime (speed) → audio replace | BROKEN |
| retime (ramp) → audio remove | BROKEN |
| stitch → audio remove | BROKEN |
| stitch → audio replace | BROKEN |
| stitch (transition) → audio remove | BROKEN |
| audio remove → trim (reverse order) | fine |
| trim → audio mix | fine |

**Ten of twelve.** And underneath them a second defect the first one had been hiding:
`trim`/`cut` never updated the compiler's `elapsed`, so `audio replace` and `audio mix`
after them had no duration bound and rendered **hours** of padded silence rather than
failing.

Every one of those verbs works perfectly alone. `vid audio remove a.mp4 | vid render` gives
a correct 8.000 s file. The 208-test suite exercised verbs singly; the nine-verb sweep above
exercised them singly; the conformance kit runs one capability. The composition — the thing
the tool exists for — was tested by nothing.

**For the spec:** it says a smart tool's capabilities are ordinary library functions and
says nothing about their composition. For any tool whose value proposition is chaining, the
combinations are a distinct surface with its own failure modes, and testing the parts is not
testing it.

---

## The Agent Skills 1024-character limit is real, breached, and checked by nothing

**Evidence: MEASURED** — parsed frontmatter `description` length for every skill the four
fleet tools ship. N=4 skills.

A real harness reported this at discovery time, not in any of our CI:

```
Skill 'fact-check'    ... exceeds 1024 character description limit (1148 chars). Continuing with discovery.
Skill 'aud'           ... exceeds 1024 character description limit (1093 chars). Continuing with discovery.
Skill 'deep-research' ... exceeds 1024 character description limit (1254 chars). Continuing with discovery.
```

| skill | chars | limit |
|---|---|---|
| `deep-research` | 1254 | 1024 |
| `fact-check` | 1148 | 1024 |
| `aud` | 1093 | 1024 |
| `vid` | 829 | ok |

**Three of four.** All three had passed the conformance kit, and the kit had said nothing,
because the limit belongs to the Agent Skills specification rather than to the Smart Tools
spec — and the file that breaches it is the one the Smart Tools spec tells you to ship.

Two things make this worse than an ordinary bug. The message says **"Continuing with
discovery"** — the host degrades rather than failing, so the tool is *partially* discoverable
and nobody is told. And it surfaces in the **host**, on a user's machine, where no tool
author is watching. Ours was caught because a human happened to paste a log.

The fix was to cut what runtime help already owns (capability enumerations, measurement
lists, pipe mechanics) and keep what does selection work: the distinctive trigger phrasings
and the non-goal clause. 1093 → 713, 1254 → 784, 1148 → 804, with every boundary intact.

**Proposal for the kit:** one deterministic rule — parse each `skills/*/SKILL.md`
frontmatter and fail over 1024 characters. It is a ten-line check against a limit that
already exists and is already being violated by most of the fleet. We added it as a
regression test in both repos; it belongs in the kit, where it would have caught all three
before release.

---

## A smart tool that shells out inherits a version surface the spec never mentions

**Evidence: MEASURED** — the same filter graph against two ffmpeg builds. N=2 builds,
1 graph.

`vid`'s zoom fix passed locally and turned CI red. Same code, same tests, same fixture:

- dev machine: `ffmpeg version N-126593-gbc46eab87c-20260916` (a nightly) — **passes**
- CI runner: `ffmpeg 6.1.1-3ubuntu5` (Ubuntu stable) — **exit 234**,
  `[Parsed_scale2ref_6] Undefined constant or missing '(' in 'rw'`

The fix had built a 52-node filter graph — a ten-way split, then per segment a trim, scale,
crop, `scale2ref`, `nullsink` and `setsar`, joined by a ten-way concat — to express one
continuous zoom. The nightly tolerated it. Stable did not.

The replacement is **one node**, by driving `zoompan` correctly instead of routing around it
(`d=1` for one output frame per input frame, `s=` to pin the size, `fps=` to give its `time`
clock real seconds, and a time-gated `z` expression):

```
[0:v]zoompan=z='if(lt(time,2.0),1,if(lt(time,4.0),1+(1.3-1)*(time-2.0)/2.0,1.3))'
      :x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=1:s=640x360:fps=25[v1]
```

52 nodes to 1. There is nothing left for two ffmpeg builds to disagree about.

**The generalisable part:** the spec's `requires[]` lets a tool declare that it needs
`ffmpeg` and where to get it. It has no way to say **which ffmpeg** — no version floor, no
required feature. A tool can be fully conformant, declare its prerequisite honestly, install
cleanly, and still fail on the user's machine because their build of the binary disagrees
with the author's. The generated-code cases are worse: `vid` generates ffmpeg filter graphs
and LUTs, so the surface it depends on is far wider than "ffmpeg is present".

We also learned the cheap discipline: **reproduce under the stable build before fixing.** A
green run on the nightly proved nothing, and that is precisely how the broken graph reached
`main`. A container with the distro's ffmpeg takes a minute and is the only honest check.

---

## A declared gate that cannot pass is a gate nobody reads

**Evidence: OBSERVED** — `.pre-commit-config.yaml` vs `.github/workflows/ci.yml`, plus a
diagnostic count on a pristine checkout. N=1 repo.

`vid` declared a `ty-check` hook in `.pre-commit-config.yaml`. It had been failing with ~50
diagnostics since before any of this work, and it was **absent from CI**, which ran only
ruff and pytest. So CI was green while the declared local gate was red, and the lesson
available to any contributor was: run `prek`, ignore the red one.

That is the actual damage. Not the 50 diagnostics — a checker nobody can satisfy trains
people to skim past a category of signal, and the next real finding in that category is
skimmed past too.

Breaking them down turned out to matter: 13 were one line (a dispatch dict handing every
handler a full union), 3 were a genuine declared-vs-returned tuple mismatch, 4 were real
(a kwargs unpack that let a config land on a `bool` parameter, two `subprocess.run`
overloads) — all fixed. The remaining 30 were `ty` being wrong about two documented Typer
idioms, and those got a scoped `[[tool.ty.overrides]]` with a written reason rather than a
blanket ignore. Then `ty check` went into CI, so it cannot rot again.

**Silence that is written down beats a red gate that is ignored.** Both are "the check
doesn't fire"; only one leaves a record of the decision.

---

## What this changes about how to call a smart tool finished

**Evidence: JUDGMENT** — argued from the eight sections above. The counter-arm (shipping on
conformance alone) is what both tools did, and its results are the rest of this document.

The ladder we actually needed, cheapest first:

1. **`check-conformance`** — 16 deterministic rules. Necessary, fast, and not close to
   sufficient. Both tools sat at 16/16 while carrying ten deviations each.
2. **`check-spec-adherence`** — the shape review. Run it **repeatedly** until a run names
   nothing new, and sweep each finding for siblings rather than fixing the named instance.
3. **Render the output and measure it.** The only step that found a 50× duration bug, a
   silent resolution change, and an index that recorded failure as success. Nothing above it
   on this ladder can see any of them.
4. **Exercise compositions, not just capabilities** — if chaining is the value proposition,
   ten of twelve combinations can be broken while every part works.
5. **Run the external binary the way a user's machine will** — a stable build in a
   container, not the author's nightly.

Steps 3 and 4 are not in the spec, not in the kit, and not in the reviewer. They found the
worst three defects in this document.

**The uncomfortable summary:** every check available to us is a check on *structure* — the
manifest, the descriptor, the process contract, the source, the plan document, the argv
string. Structure was fine in all three of the worst cases. `d=60` is well-formed. A
single-shot index is well-formed. An unconnected filter pad is a well-formed plan compiled
to a well-formed command. The tool was wrong about the world, and only the world could say
so.

---

## Round two: the same ladder on two research tools, and what step 3 means when there is nothing to render

**Evidence: MEASURED** — both creator checks plus a real-spend output sweep on `deep-research`
and `fact-check`. N=2 tools, 2 reviewer runs each, 1 correctness sweep (~$0.53 of real API
calls). Raw reports and 42 artifacts in `evidence/colombod/`.

The same ladder, applied where the deliverable is a *brief* rather than a rendered file.

**The reviewer could not run at all.** `requires-python = ">=3.11"` was declared in all three
pyprojects while `research-core[agent]` pulls a dependency needing `>=3.12` — and that extra
is not optional, it is each tool's only dependency line. So `uv run` failed outright in both
tool roots, which is exactly how the creator's harness stands a tool up. Three CI jobs were
green anyway: CI runs the kit from the *spec's* checkout, so the tool's own project is never
synced, and `uv tool install` resolves against one concrete interpreter rather than the
declared range. **A declared version RANGE is only tested by something that resolves the
range.** Nothing did.

**Then: 6 adhere / 10 deviate, each.** A third and fourth tool at ten. The reviewer's
sampling behaviour reproduced exactly — round two returned 6 and 5 remaining, the classes
recurring against new sites the first run had not read.

**Step 3 here is not rendering — it is asking whether the answer is sound.** It found eight
defects, and the worst is the direct analogue of the 50× video bug:

- **Nothing in either tool had ever fetched a cited source.** The brief's load-bearing
  citation — the one it named as *why* its confidence was "high" — returned HTTP 404. It was
  published unmarked in the brief, the sources view and the bibliography, and then underwrote
  four "high"-confidence verdicts one hop downstream. The answer happened to be correct and
  another citation genuinely supported it, which is the point: **nothing in the system could
  tell the difference**, and no amount of reading the source would reveal it.
- **`fact-check verdicts` always returned `tally: null`** in a document whose own contract
  says the tally *is* the answer — reader keyed `tally`, writer wrote `counts`. It survived
  342 tests because the test asserted against a **hand-written fixture** carrying two keys no
  engine has ever written and missing five that every engine writes. That is the
  never-hand-write-a-mock rule earned for the fourth time, in its fourth repo.
- **A failed run under-reported its cost 13.8×** — $0.027 claimed, $0.344 burned — because the
  discarded-cost arithmetic hangs on a value only constructed when a stage *succeeds*. The
  field that exists to report wasted money was structurally absent from every run that wasted
  any.
- **Two defects that existed only under `--detach`**: a relative file path worked attached and
  always died detached; a nonexistent run id was accepted detached and refused attached. Both
  are the detached path skipping validation the attached path does — composition again, one
  layer down.
- **The repair loop fed back the wrong finding three times.** Three rejected replies all failed
  on one unescaped quote; the feedback said "no JSON document, no prose around it", so the
  retries added a code fence and kept the bug. $0.34 to make one mistake three times.

**What the composition test found, for once, was that it works.** `fact-check --from-run`
transferred all nine source ids and URLs byte-identical with zero dangling citations. Worth
recording as a negative result: the seam the video tool got wrong, this one got right.

**A sixth rung, learned the hard way.** The 0.10.0 release left `main` RED twice. A version
bump here touches seven files — three pyprojects, three manifests, two generated skill files
— and the checks that catch a half-done bump live only in CI. Same shape as the ffmpeg
divergence: *a gate CI runs that the local loop does not*. Closed the same way, with a
`preflight.sh` proven by reproducing the historical failure rather than asserted. So:

6. **Run what CI runs, before you push, and prove your harness by making it fail on a commit
   that already failed.** A parity harness that cannot reproduce a known divergence is
   theatre.

---

## Ledger

All numbers reproducible from the linked repos.

| tool | before | after |
|---|---|---|
| `aud` conformance | 16 PASS / 0 FAIL | 16 PASS / 0 FAIL |
| `aud` spec adherence | 7 adhere / **10 deviate** | **0 deviate** |
| `aud` tests | 418 | 420, 0 skips |
| `vid` conformance | 16 PASS / 0 FAIL | 16 PASS / 0 FAIL |
| `vid` spec adherence | 7 adhere / **10 deviate** | closed across three rounds |
| `vid` tests | 175 | **221**, 0 skips locally |
| `vid` zoom, 8 s source | **400.000000 s**, 320x180 | 8.000000 s, 640x360 |
| `vid` zoom filter nodes | 52 | **1** |
| `vid` composed audio chains | **10 of 12 broken** | 12 of 12 render |
| `vid` `ty check` | ~50 diagnostics, absent from CI | clean, in CI |
| skills over the 1024-char limit | **3 of 4** | 0 of 4 |
| `deep-research` spec adherence | 6 adhere / **10 deviate** | 6 deviate -> closed |
| `fact-check` spec adherence | 6 adhere / **10 deviate** | 5 deviate -> closed |
| research tests | 295 | **363**, 0 skips, green scrubbed |
| `uv run` in either tool root | **unresolvable** | works |
| cited sources ever fetched | **never** | `sources --verify` |
| `fact-check verdicts` tally | **always null** | computed from disk |
| failed-run cost reported | **$0.027 of $0.344** | every rejected attempt |

Commits: `aud` [`3f70a3a`](https://github.com/colombod/amplifier-smart-tools-audio),
`vid` [`bfeaa13`, `08d8553`](https://github.com/colombod/amplifier-smart-tools-video),
`deep-research`/`fact-check`
[`a38b8a3`](https://github.com/colombod/amplifier-smart-tools-research). CI green on all.

---

## What I could not settle

- **Whether `check-spec-adherence` ever converges.** Three runs, three non-empty results,
  each finding real new instances. I stopped at three because the tool shipped, not because
  a run came back clean. Someone should run it five or six times on a stable tool and find
  out whether it terminates or just keeps sampling. That is a real open question about the
  reviewer, not a complaint about it — every finding it produced was accurate.
- **Whether a generic output-correctness rule is possible for the kit.** I argue above that
  it probably is not, because the kit cannot know what a tool is for. I did not try to
  design one, so treat that as a hunch rather than a conclusion.
- **What a version floor in `requires[]` should look like.** The gap is measured; the
  design is not. A `minimum_version` field is the obvious shape and I have not built it, so
  it is worth exactly what an unimplemented spec opinion is worth.
