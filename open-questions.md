# What we did not settle

The other files say what we found. This one says what we **still do not know**, what it
would take to find out, and which of it is worth anyone's next day.

Ordered by how cheaply a real answer could be had, not by how interesting the question is.

---

## Ready to test tomorrow

### 1. Does an intent-shaped description actually get picked up?

**Status: FIXED, NOT MEASURED.** This is the honest gap in our loudest finding.

All three of our tools shipped skill descriptions that matched the tool's **name** rather
than the user's intent, and a real user in Codex had to name the tool explicitly every
time. We rewrote all three to open on quoted user phrasings. We have **not** verified that
the rewrite works.

**What would settle it:** the same harness, the same task, phrased without the tool's name
— *"trim this down to where she explains the pricing"*. If it still needs prompting, the
matcher wants something structural and better prose will never fix it. **A negative result
here matters more than the rewrite did.**

### 2. Can a conformance kit check *reachability*?

Two of our tools passed 15/15 while carrying a description no user phrasing could reach,
and `use_cases` that was a single sentence copy-pasted from `description`.

**Two candidate checks, both lints rather than judgment calls:**

- `use_cases` must not duplicate `description`
- warn on a description whose only hook is a keyword list or the tool's own name

**What would settle it:** implement both against the catalog and count how many existing
tools trip them. If it is most of them, the checks are right and the convention is missing.

### 3. Does the "deterministic check" rule generalise?

Our strongest claim: **a model-backed capability is safe to ship unattended when a cheap
deterministic check exists on its output.** Three instances in one tool — a generated
ffmpeg expression sampled mid-blend, a narration line measured against its slot, a colour
transfer scored by statistical distance.

Three instances in **one tool by one author** is not a law.

**What would settle it:** take three model-backed capabilities from tools nobody here
wrote, and ask of each — is there a cheap check on the output, and does the tool run it? If
the ones with checks are the ones people trust, the rule holds.

---

## Needs a schema change, so needs agreement first

### 4. `model_backed` is a boolean; real tools are plural and tiered

One of our tools uses **four kinds of intelligence**: speech-to-text and text-to-speech
**locally with no credential**, plus text reasoning and vision remotely. A consumer asking
*"what AI does this use?"* gets a four-row table, and two rows need no credential at all.

`requires[]` cannot say **which capability each entry unlocks**. The only place that mapping
exists in our tools is prose we wrote by hand.

**What would settle it:** a proposed shape, tried against a tool that has exactly one model
capability and one that has four. If the same schema serves both without ceremony, it is
right.

### 5. `requires[]` names a binary and cannot describe it

A real macOS install found all three of these missing at once:

- **which build features** of a binary you need — `ffmpeg` is necessary and not sufficient;
  `caption` needs it built with libass
- **that an install may need a postinstall step** — `brew postinstall ca-certificates
  fontconfig gnutls glib openssl@3`, or captions render with no text in them
- **that only one capability breaks** without it — every other verb was fine

All three were true, none expressible, so the tool carries them in prose.

### 6. Should the contract say vision is already possible?

`AgentRequest` has no image field, so the natural conclusion is that the seam cannot do
vision. **It can** — the request carries a `workspace`, and the agent behind it has
file-reading tools. We verified it before building on it.

A builder reaching the natural conclusion would fork the seam, add a field, or go straight
to a provider SDK — all of which break the boundary the scaffold exists to create.

**What would settle it:** one sentence in the interface docstring. The question is only
whether anyone else has already hit this and solved it differently.

---

## In flight — being tested by building

### 7. What does it mean for one smart tool to depend on another?

**Status: IN PROGRESS. Nothing built yet, so nothing here is a finding.** Recorded now
because the design questions surfaced before the code, and the answers should be written
against what building actually does rather than reconstructed afterwards.

This is **ROADMAP q3** ("how other products consume smart tools") from an angle the question
does not obviously anticipate: not a *host* consuming a tool, but a **tool consuming a tool**.

**What is being built:** `audio-mix`, a multi-track mixing and spectral-ducking tool that
depends on `aud`
([amplifier-smart-tools-audio](https://github.com/colombod/amplifier-smart-tools-audio)).
The dependency is deliberately **both** kinds at once, which is what makes it interesting:

- **internal** — `audio-mix` imports `aud` as a library inside its own verbs;
- **compositional** — the user chains at the invocation level: `aud` (process stems) →
  `audio-mix` (mix) → `aud` (master).

So `aud` sits **before and after** `audio-mix`. A sandwich, not a chain.

**Evidence: OBSERVED** — source read of `aud` at commit `818ac7d`, file:line below. No
measurement yet; nothing has been built against it.

The first concrete obstacle is already visible in the source, and it is not the one we
expected. **`aud` has no promised library API.** It promises `contracts/plan.v1.md`,
`contracts/regions.v1.md` and its CLI surface — its own docs say *"the contract document is
the interface; CLI flag spellings are ergonomics, not promised."* The `dsp/` boundary rule
in its `AGENTS.md` §8 (arrays in, arrays out, no config, no filesystem) is a **contributor
design rule**, not a stability guarantee to an importer.

Plenty of names are public by spelling — `compress` (`dsp/dynamics.py:109`), `gate`
(`dsp/gate.py:310`), `brickwall` (`dsp/limiter.py:85`), `detect_transients`
(`dsp/detect.py:169`), `resolve_points` (`dsp/resolve.py:368`), `crossfade`
(`dsp/edit.py:74`). But *importable* is not *promised*. And several pieces the dependent
tool specifically wants are private: `_static_gain_reduction_db` (`dsp/dynamics.py:73`),
`_gate_curve` (`dsp/gate.py:110`), `_smooth_gain_db` (`dsp/dynamics.py:94`),
`_hold_attack_release_db` (`dsp/gate.py:177`), `_align_to_zero_crossing`
(`dsp/resolve.py:148`).

**Evidence: JUDGMENT** — argued from the above, not measured. The other arm was not built.

The spec says **the library is the tool** and the CLI is a thin adapter over it. But the
manifest, the descriptor and the conformance kit all orient around the CLI and the manifest.
Nothing asks a tool to declare its **stable import surface**. If tools are meant to depend on
each other as libraries — which the spec's own framing invites — something has to say which
imports are promised. The moment a sibling imports `aud.dsp`, `aud` loses the freedom to
refactor it, and today that freedom is unpriced because nothing external depends on it.

### Corrections and confirmations — spec read, 2026-09-24

**Evidence: OBSERVED** — specs and source read at pinned revisions:
`microsoft/amplifier-smart-tools@61e59d5`, `microsoft/amplifier-smart-tools-catalog@7f6379e`,
`vercel-labs/skills@7407f38`, `astral-sh/uv@3db6652`. Nothing installed, nothing run.

**CORRECTION — we are not first, and the "first case" framing above was wrong.**
`showrun` (`robotdad/amplifier-smart-tool-showrun` 0.1.0) already declares the catalog tool
**Stories** as an optional `requires` entry. It does **not** take it as a package dependency:
the caller passes a path to a separate interpreter (`--python /absolute/stories/bin/python`),
and the version constraint lives in the entry's `purpose` **prose** because the schema has
nowhere else to put it. So `audio-mix` would be the **second** catalog tool to depend on
another at all, and the **first to depend on one as a library/package**. Keep the distinction;
the weaker claim is the true one.

*(Non-qualifying near-cases: `fact-check` mentions `deep-research` but its real dependency is
`research-core`, a shared library rather than a smart tool. `tmux`'s `requires: tmux` is the
binary.)*

**CONFIRMED — Agent Skills has no skill-to-skill dependency mechanism.** The frontmatter is a
**closed set of six fields**: `name`, `description`, `license`, `compatibility`, `metadata`,
`allowed-tools`. The reference validator enforces it —
`ALLOWED_FIELDS = {...}`, and anything else errors with *"Unexpected fields in frontmatter"*.
So **inventing a `requires:` key would fail `skills-ref validate`**. `metadata` is explicitly
semantics-free. `npx skills add` resolves nothing transitively (grepped `add.ts`, `skills.ts`,
`installer.ts`, `skill-lock.ts`, `update.ts` — no dependency concept anywhere), though one
command *can* install several skills found in one repo.
=> The prose-only companion note is not a stopgap. It is the **only** conforming option.

**CONFIRMED — `description` cap is 1024, and host behaviour past it genuinely diverges.**
The spec's own lenient-validation guidance covers an over-length *name* and a *missing*
description but **says nothing about an over-length description**. Observed: `skills-ref`
errors; Amplifier warns and loads anyway; `npx skills` does not check; one third-party report
says Pi drops the skill. A guard test is the right call precisely because the answer is
host-dependent.

**CONFIRMED — `uv tool install X` does not expose a dependency's executables.** Verbatim from
uv's docs: *"Executables provided by dependencies of tool packages are not installed."* The
documented fix is `--with-executables-from`, which differs from `--with` exactly in this.
=> A smart tool that shells out to a sibling's CLI **does** produce a silently broken install
under the ordinary install command.

**NEW HAZARD, not previously on our radar.** The name `aud` on PyPI is **someone else's
package** (`zdhoward/aud` 2.0.1, a different audio tool). Any bare `aud` requirement that
falls back to PyPI installs the wrong project. Compounding it: `tool.uv.sources` is
**uv-only** — *"Sources are only respected by uv"* — so a non-uv consumer resolving
`dependencies = ["aud"]` gets the wrong package. A direct PEP 508 git reference avoids it.
And a git-URL dependency means `audio-mix` **could never be published to PyPI** as-is.

**The spec ambiguity that has to be resolved by whoever builds this.** `requires` is for
environment prerequisites, and the manifest spec says plainly: *"Language runtimes and
package dependencies are not listed here. Those belong to the packaging system, which already
resolves them."* But `aud` is **both** a package and a smart tool. Import it as a library and
that sentence says it does not go in `requires`; shell out to it and it is an environment
prerequisite that does. The spec does not say which rule wins. `invocation.md` leans one way
without ruling: *"Capabilities… within one tool or across several, compose by writing a script
against the libraries, where results are typed return values rather than text to parse."*

**The conformance kit cannot express any of this — three concrete refusals, read from the
rule code.** `manifest-fields-closed` would **FAIL** a new top-level `depends_on`.
`manifest-requires-shape` is `extra="forbid"`, so adding `version:` or `kind: smart-tool` to a
`requires` entry **FAILS**. And `install:` **FAILS** if it begins with `uv`/`pip`/`npx`… or
contains whitespace, so `install: uv tool install git+…` is rejected by design. A dependent
tool declared purely through `pyproject` passes every dependency-related rule **vacuously** —
no rule reads package dependencies at all. The kit also *"never installs the tool under
test"*, so if the CLI is absent all six runtime rules **SKIP** rather than FAIL.

**Named experiments still open — these need running, not reading:**

1. **Run the kit against `audio-mix` once it exists** and record which rules pass vacuously.
   The non-negotiable is *"deterministic capabilities work with zero provider credentials"* —
   it says nothing about **zero sibling tools**.
2. **Does `--with-executables-from` accept a git-URL requirement**, and is it safe against
   PyPI's `aud`? Undocumented; needs a container run.
3. **Is a transitive dependency's CLI reachable via `uv run`**, or from inside a `uv tool`
   environment? uv's docs never say. Needs a container run.
4. **Where does a minimum sibling version live?** Nothing in the manifest schema can carry
   one. `showrun` put it in prose. Is prose the convention, or the gap?

---

## ROADMAP questions we did not touch

Stated plainly so nobody assumes they were covered. **q3 is no longer on this list** — see
§7 above, in flight.

| # | question | our position |
|---|---|---|
| 5 | **Continuing a smart call** — clarification, session continuation | **No evidence either way.** We never needed it. That is weak evidence it is unnecessary, and we may simply have built tools that do not want it. |
| 6 | **Generated wrappers** — an SDK or MCP wrapper over an existing tool | **Untouched.** A generator over one of ours would be a day's work and would test whether the manifest carries enough to generate from. |

On **#1 (long-running)** we have a position and an implementation — `--detach`/`--wait`,
with in-flight runs deliberately not durable past the calling session. Whether those should
be **standard verbs in the spec** is still open, and is a decision for the spec's authors
rather than a thing we can settle by building.

---

## Things we would explore if the hackathon ran another week

**A tool nobody here wrote, taken end to end.** Every finding in this repo comes from tools
written by their own finder. The failure modes we cannot see are the ones that only appear
when the author is not in the room.

**Cross-platform, seriously.** Our DTU proved the install works on Linux and told us
nothing about macOS — a user found the real gap in about a minute. A container per platform
is cheap; we did not do it, and it is the single highest-yield thing on this list.

**Whether a smart tool should ship evaluation at all.** We built one and it stayed thin.
Open: is a per-tool eval harness the author's job, the kit's job, or nobody's?

**The negative results are worth as much as the positives.** Three from this build, all
documented in the findings and all worth repeating deliberately:

- a "works without a provider" test that passed because the environment leaked a credential
  store
- a no-ffmpeg suite that ran **with** ffmpeg, because the scrub was a guess about which
  system directories were safe
- a flat-grey fixture that reported two *opposite* colour grades as byte-identical

Each was a check that passed for a reason other than the property holding. If there is one
convention worth arguing for beyond the spec, it is that **a negative test must prove the
absence it depends on** — and none of our three did until something outside our machine
said so.
