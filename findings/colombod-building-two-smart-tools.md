# Findings: building two smart tools from an Amplifier bundle

**Who:** colombod
**What this came from:** converting `amplifier-bundle-perplexity` into two conforming smart
tools — `deep-research` and `fact-check` — in
[colombod/amplifier-smart-tools-research](https://github.com/colombod/amplifier-smart-tools-research).
Both distribution roots pass the conformance kit 15/15 with no credentials configured, and
CI re-proves it on every push.

**Status:** current as of the second milestone. The agent-integration section is install-time
evidence only; it is marked where that matters and wants revisiting once the reasoning turns
actually run.

Everything here was learned by reading the spec and the two reference implementations, or by
building against them. Where a claim rests on something you can check, the check is named.

**Every section below opens with its evidence tier.** We added these after noticing that nine
of eleven sections carried no measurement and yet read with the same authority as the two that
did — the exact failure our own tools exist to prevent, committed in the document arguing for
preventing it. The tiers:

| tier | means |
|---|---|
| **MEASURED** | a before/after or A/B we actually ran, with the figures and their sample size |
| **OBSERVED** | something we read in source or watched happen; true, but not a measurement |
| **JUDGMENT** | an argued call. The reasoning is shown; the other arm was not built |
| **PROPOSAL** | a design we have not implemented or measured. An argument, nothing more |

Read a PROPOSAL as an invitation to disagree. Read MEASURED with its sample size attached —
almost all of ours are **N=1 per arm**.
---

## What we read

**Evidence: OBSERVED** — read from the spec, the ROADMAP and the two reference implementations' source. Nothing measured; nothing needs to be.

| Source | Depth |
|---|---|
| `amplifier-smart-tools` spec + conformance kit | full, including every rule in `run.py` |
| `amplifier-smart-tool-tmux` | full source |
| `amplifier-smart-tool-digital-twin-universe` | full source |
| `amplifier-smart-tools-catalog` | 9 manifests (the catalog stores manifests only, never code) |
| `amplifier-bundle-perplexity` | full source — the bundle we are converting |
| `amplifier-bundle-dot-runner` (Attractor) | engine source, for the orchestration question |

Worth knowing: the catalog stores an exact-byte snapshot of each tool's `SMART_TOOL.md`
plus a provenance pointer, and never code or `smart-tool.json`. Reading catalog entries
tells you what tools claim, not what they do. Only the two reference repos and the spec's
own `sample-good` fixture have inspectable code.

---

## Gap 1 — nobody handles a large result

**Evidence: OBSERVED** — read from both reference tools' source. This is an absence we looked for and did not find, not a measurement of anything.

**Not on the ROADMAP.** The spec is silent, and no example addresses it.

Everything observed falls into three patterns:

| Pattern | Where | What it does |
|---|---|---|
| refuse over ceiling | tmux `read --lines`, hard `MAX_READ_LINES` | refuses rather than silently capping |
| bound and say so | tmux's completeness block | returns a slice with `complete: false` and a note naming what was left out |
| write if asked | DTU `--out` | writes an artifact and returns the path — caller opt-in, not size-driven |
| refuse over bytes | team-pulse, 200,000-byte ceiling | named exception, override flag, documented narrower alternative |

None answers "the result is inherently large and the caller did not ask ahead". Nothing
writes to disk on the basis of size, and nothing returns a navigation path with the
result.

**What we propose:** a capability whose result may exceed a threshold returns an
identified artifact plus a real summary and a narrowing command, never a silent
truncation; any partial view states that it is partial. The navigation hints travel *in
the result*, so a caller that has never seen the tool learns to consume an oversized
result from the result itself.

Nothing in the spec or the conformance kit penalises this.

---

## Gap 2 — nobody handles a long-running call

**Evidence: OBSERVED** — as Gap 1. Our own long-call handling below is built, but the claim *here* is about what the reference tools do.

**ROADMAP #1.** Acknowledged open, unaddressed by every example.

- tmux pins the engine's display to `verbosity="quiet"` and wraps exactly one turn in a
  180-second timeout. Its own `contracts/cli.v1.md` backlogs streaming in writing and says
  it expects to hit the question first and feed evidence back.
- DTU blocks synchronously; its roadmap lists structured streaming progress as not built.
- The closest anything gets is a bounded step or turn ceiling **reported after the fact**
  — music-deck's turn ceiling, team-pulse's six-step bound, home-assistant's 120-second
  budget with defined partial states. None emits anything during the call.

**What we propose:** newline-delimited JSON progress on stderr — which the spec already
reserves for diagnostics and already names progress as a category of — and, critically,
**the same records persisted to the run's own event log**. A pure stream is lost to anyone
who was not watching; a persisted record can be reconstructed afterwards. Detached mode,
returning a handle to poll, is the stronger version and is in our backlog rather than v1.

---

## Divergences worth knowing

**Evidence: OBSERVED** — read from source, side by side. Where the two reference tools disagree is a fact about them; what the spec *should* say about it is our opinion.

**The output envelope is not one convention.** `sample-good`, tmux and DTU all use
`{"result": ...}` / `{"error": {"code", "message", "remedy"}}` — three independent
authors, same shape, which makes it the de facto standard. team-pulse returns bare domain
JSON with no wrapper. The perplexity bundle returns markdown prose aimed at an LLM reader.
Pick deliberately based on who the caller is; we took the strict envelope.

**Credential-absence is enforced at different points.** Every catalog manifest claims a
per-verb runtime check; the perplexity bundle instead swaps in a placeholder at mount
time and never fails to mount. The conformance kit is built around the per-verb model, so
the bundle's pattern would not pass.

**The catalog does not enforce the spec's own definition.** `structure.md` requires a
smart tool to have a genuinely model-backed capability. The gmail and spotify entries have
none and are catalogued anyway.

**Configuration is another split, and the spec says nothing.** DTU states flatly that
"every setting is an environment variable; there is no config file". tmux-fleet has a
real config file at `~/.config/tmux-fleet/config.json`, its path overridable by
`TMUX_FLEET_CONFIG`, resolved **above** the environment variable on the reasoning that a
deployment which wrote a config is entitled to have it honoured while an environment
variable can arrive by accident from a parent process. It also distinguishes absent from
explicitly-null from wrong-type — the third being fatal rather than a silent fall back to
the default — and reports the provenance of the value it chose, including ambient settings
it saw and deliberately did not honour.

tmux's model is markedly better and nothing in the spec asks for either. Worth raising:
a smart tool is called by hosts that inject environment variables freely, so "which tier
won, and why" is not a nicety. We are adopting tmux's model and adding a `config` verb
that reports provenance per setting.

**"Library-first" is asserted more than demonstrated.** Four manifests use the phrase "one
library, one thin CLI". The only fully readable fixture, `sample-good`, has no separate
library module at all — `cli.py` *is* the library. The principle is well argued in the
spec; the worked multi-file example is in tmux and DTU, not in the fixture.

---

## Attractor — assessed, declined

**Evidence: JUDGMENT** — a design decision with its reasons written down. We did not build the other arm, so this is an argued call and not a finding.

The `attractor-expert` agent fails to load in this workspace
(`system-attractor-expert.md` missing from the dot-runner bundle cache), so the engine
was read directly instead.

**Embeddable, but through a door the docs do not point at.** The documented seam,
`run_pipeline()`, always constructs a Foundation bundle and session and, on first call
in-process, reaches the network and shells out a package install into `~/.amplifier/`.
That breaks the "no Amplifier runtime assumed" constraint outright. The lower-level
`drive_engine(coordinator=None, default_worker="llm-direct")` genuinely works with no
bundle, session or coordinator — the engine's own test suite exercises exactly that — but
every README in the repo documents the former. Unlike the agent library, nothing in the
package mutates the environment at import.

**Declined for three reasons:**

1. `llm-direct` is bare model calls, not a tool-using agent. Research needs web access,
   which means the spawn worker, which means a synthesized bundle and a spawned session —
   back inside the runtime we cannot assume.
2. Fan-out width is fixed at parse time: the parallel handler fans out over edges declared
   in the `.dot` text. Fact-checking has N claims, N unknown until runtime. That means
   generating graph text per run — a code generator driving a graph interpreter, to
   express what a bounded `gather` already says in one line.
3. Neither pipeline branches. A four-node linear graph is a for-loop; adopting a format,
   parser, validator and a large graph-walking engine to run it means owning two failure
   domains instead of one.

**Two things taken anyway:** its checkpoint discipline (write after every node, carry a
graph fingerprint, refuse loudly to resume a missing, corrupt, wrong-schema,
already-completed or mismatched checkpoint) and its event vocabulary (`node_start`,
`node_complete` with duration, `checkpoint`, `error`, `complete`).

> ★ **Insight:** the engine was buildable-with and still wrong, because our pipelines have
> no branching and a fan-out width nobody knows until runtime — the exact two things a
> graph engine exists to handle. Principle: adopt an orchestrator when the control flow is
> the hard part; when the control flow is a for-loop, the orchestrator is the hard part.

---

## What building it actually taught us

**Evidence: MIXED** — most items are OBSERVED (what the code did when we ran it, including live runs with figures quoted inline). The generalisations drawn from them are JUDGMENT. Each item names which it is where the distinction matters.

Everything above came from reading. This section came from building, and it is the part
no amount of reading produces.

### Scripted tests bless what one live call breaks

Our suite was green, both distribution roots were conforming, and every model-backed path
was exercised through the backend seam with no credentials. The first real call against
the research service broke three things at once:

- The service emits `[web:1]`, not `[1]`. Our citation rewrite matched neither, so **every
  marker in a live report stayed unrewritten** — and the dangling-citation check, the
  sharpest thing in our validation layer, had nothing to check. It was passing because it
  was looking at markers that did not exist.
- The result's navigation block promised `read <id> --sections 1-3` for a three-line report
  with no sections. Running it exited 2.
- `raw/` held a single repr string, because `json.dumps(default=str)` cannot serialise an
  SDK model object. The verbatim record was neither verbatim nor replayable, defeating both
  reasons the directory exists.

**The generalisable rule, and the one we would most want in the spec:** any output that
tells a caller what to do next must be *derived from the state that exists* and *executed
by a test*, not produced from a template. A navigation hint that does not work is worse
than no hint, because the caller stops trusting the whole block.

### The library-first rule has no conformance check, and we failed it for three milestones

The spec calls it absolute — *"no capability exists only in a wrapper"* — and nothing
checks it. Our CLI exposed eleven verbs; the tool's own library exposed two. Nothing was
missing: the pieces were all in the shared package. The capability was only ever
**assembled inside the wrapper**, so a caller doing `import deep_research` got two
functions and no hint the other nine were reachable under a different name.

Worth noting that `sample-good`, the only fully readable fixture in the spec repo, does not
demonstrate a separate library either — `cli.py` *is* the library there. The principle is
well argued and nowhere exemplified.

**Proposal:** either make it checkable, or publish the test pattern we ended up with — read
the verb table off the real parser, and assert every verb has a library equivalent whose
signature takes no CLI machinery.

### `requires` declares, but nothing asks whether the tool detects

The spec is explicit that detection is the tool's own job, and then never returns to it.
Nothing in the conformance kit asks whether a tool can actually tell you if its
prerequisites are present.

We built `check` driven *by* the manifest rather than by a second hard-coded list, with one
rule worth naming: a requirement with no detector reports **`unknown`, never `satisfied`**.
The drift is asymmetric — forget to declare something and you merely under-promise; forget
to detect a declared thing and the tempting default is a false all-clear for someone whose
run is about to fail.

**Also cheap to add:** `install` is checked not to be a command, but nothing checks the
document it points at exists. A dangling pointer is handed to someone who is already stuck.

### How does a smart tool find its configuration? Nothing says, and the two examples disagree

This is the largest unaddressed gap we hit, and unlike the others it is not a rule that
goes unchecked — there is no rule.

**The spec is silent.** Searching the whole of `spec/` for configuration guidance returns
nothing: no config-file location, no precedence, no environment-variable naming convention,
no XDG mention. Every occurrence of "configured" is about *whether a provider's credentials
exist*, never about *how a tool discovers any of its settings*.

**The two reference tools resolve it differently, and neither documents it in its manifest.**

| | tmux-fleet | digital-twin-universe |
|---|---|---|
| Config file | `~/.config/tmux-fleet/config.json`, path overridable by `TMUX_FLEET_CONFIG` | **none at all** |
| Settings | file, with env override | environment variables only (`AMPLIFIER_DTU_PROVIDER`, `AMPLIFIER_DTU_MODEL`, `AMPLIFIER_DTU_MAX_ENVIRONMENTS`) |
| Declared in `SMART_TOOL.md`? | credentials mentioned; the config file is **not** | configuration **not mentioned** |

So a caller installing two conforming smart tools must discover two unrelated
configuration systems by reading source, and the manifest — the one file the spec designates
for telling a caller what a tool needs — describes neither.

**What we did, and the wall we hit.** We took tmux's shape (argument > config file >
environment > default, with config deliberately *above* environment) and inverted it for
credentials only (environment first, then a 0600-fenced file). Then TOML bit us: tmux uses
JSON, which has `null`, so it can distinguish absent from an explicit "no opinion" from
wrong-type. **TOML cannot express null**, so that trichotomy collapses to two cases. The
tempting fix — `depth = ""` as a stand-in — destroys the distinction it was meant to
preserve, since an empty string becomes simultaneously "no opinion" and "a wrong value". We
corrected our documentation rather than fake the capability.

**The related question nobody has answered: may a smart tool read its host's config?**

Concretely: Amplifier keeps provider credentials in `~/.amplifier/settings.yaml`. A smart
tool invoked from an Amplifier session could read it and spare the user configuring the
same key twice. We observed the boundary holding by accident during a container test — an
Amplifier session was model-backed while `deep-research check` inside it correctly reported
`ai-provider: absent`, because the key lived in the host's settings file and the tool reads
only the environment and its own credentials file.

**And it turns out it is already happening, invisibly, in one direction.** With every
`*_API_KEY` scrubbed from the environment, our embedded engine still reported `anthropic`
usable — because the engine IS the host's own library (`amplifier-agent`) and resolves the
host's credentials directly. So the agent-backed paths of our tool piggyback on Amplifier's
configuration automatically, whether we asked or not, while the Perplexity path does not
and cannot. Our own `check` verb was reporting `ai-provider: absent` about a path that
works, because it asked our environment instead of asking the thing that runs the turn.

That asymmetry is worth the spec's attention on its own: **embedding a host's agent library
silently inherits that host's credential resolution.** A tool author who reasons about
"my configuration" will get this wrong, in both directions — claiming absent when a turn
would run, and being surprised when a credential they never configured turns out to work.

We think **automatic** piggybacking we *choose* is wrong and that the spec should say so:

- A smart tool is host-agnostic by definition — *"consumable anywhere: Copilot, Claude Code,
  a Python service, a shell script"*. Reading one host's private config file makes it that
  host's tool.
- Silently inheriting a credential from another application's file is a security surprise.
  The user configured that key for Amplifier, not for whatever Amplifier happened to invoke.
- It is a private format belonging to another project, free to change without notice.

**Explicit opt-in** is the defensible middle for the part we control: a setting a user
deliberately turns on (`host_config`, unset by default) that says "also look in this host's
configuration". It removes the double-configuration annoyance without making the tool
secretly host-coupled. What we could not opt out of is the engine's own resolution — that
comes with embedding it, and the honest response was to make `check` report where a
credential actually came from rather than pretend the tool's own environment is the whole
story.

**What would settle it.** Any of these would be an improvement on silence: a named
convention (`~/.config/<tool>/config.toml` plus `<TOOL>_CONFIG`), a required precedence
order, a rule that a tool's configuration surface be declared in its manifest the way
`requires` declares prerequisites, or an explicit statement on host-config piggybacking.
We have a working implementation of the first two and would happily donate the shape.

### Two conventions now have working evidence rather than a proposal

- **Large results:** a real run returns a 220-byte envelope pointing at a run directory,
  with `inline` flipping on a byte threshold and every `next` command verified to execute.
- **Long-running calls:** progress streamed to stderr as newline-delimited JSON *and*
  persisted to the run's own event log, confirmed on live calls. Also concrete: the service
  reported no cost, so `usage.cost_usd` is `null` rather than a fabricated `0.00` — the
  honest-unknown convention with a real instance behind it.

### Integrating two kinds of intelligence: the differences that mattered were not interface differences

We ended up with two seams — one that acquires evidence (a research service, or an agent
with web tools) and one that reasons over evidence already gathered (an agent with **no**
tools). At arm's length they are the same shape: an intent in, a result plus a usage
account out. We spent a design note deciding whether to collapse them into one `Agent`
abstraction with a `can_search` capability flag, and **decided not to**.

The reason is the transferable finding. **They do not differ in shape; they differ in
authority** — in what each is permitted to introduce into the caller's world. A backend may
bring new sources into a run. A reasoner may not, and that is the entire premise the
citation check rests on: a synthesis citing `s9` when the run defines `s1`–`s3` is rejected
and repaired, and the check only means anything because the synthesising turn *could not
have found* `s9` itself. Merge the two and the result type grows an optional `sources`
field — which is precisely the route by which a synthesis quietly introduces a citation
nobody gathered. The type system makes that unrepresentable today; a capability flag would
demote it to a configuration mistake, and configuration mistakes ship.

**For ROADMAP #2 and #4:** an interface specification that describes only the call
shape — arguments in, structure out — will produce tools that compile and lie. The
load-bearing question when integrating intelligence is *what is this implementation
allowed to introduce*, and that is not visible in a signature.

**A second finding for #4, and it begins with us being wrong.** We previously reported here
that cost is not uniformly reportable — that one of our two backends simply could not tell
us what a call cost, on the strength of `cost_usd` coming back `null` on live runs.

**That was our parsing bug, not the service's limitation.** Perplexity reports spend in
*more* detail than our agent backend does:

```
cost.total_cost               0.0166 USD
cost.tool_calls_cost_details  {fetch_url: 0.01017, search_web: 0.0025}
```

plus input, output and cache costs separately, and invocation counts per tool. We were
reading a flat `cost_usd` field that does not exist on that response shape, finding nothing,
and recording `null`. Every call was invisible in our own accounting while the caller was
genuinely being billed — and a comment in our code explained that `null` "means unreported",
documenting our own bug as a property of somebody else's service.

**What survives is a better finding.** The real variable is not *whether* spend is
reportable but **when it becomes knowable**. An agent running inside a turn we watch reports
per call, as it happens. A service that owns its own search loop can only account
afterwards, in one block. An interface modelling only a final total cannot express live
spend; one modelling only a stream cannot express a service that answers in one shot. Both
of ours now replay their work as the same kind of tool event, so a caller sees the searches
and fetches either way.

**And a sharper lesson than either, which we would offer to the conformance kit:** an
integration should verify it can read a provider's *accounting* the same way it verifies it
can read the *answer*. We had a test asserting `null` meant "unreported". It passed
throughout, on a value produced entirely by our own bug. A tool that cannot see what it
spends is not merely untidy — it cannot honour the spec's own requirement to fail loudly
about cost, because it does not know.

The full argument, including the strongest case against our decision and why
reversibility settled it, is in the tool repo:
[`docs/DESIGN-NOTE-one-abstraction.md`](https://github.com/colombod/amplifier-smart-tools-research/blob/main/docs/DESIGN-NOTE-one-abstraction.md).

---

## Friction on the paved path

**Evidence: OBSERVED** — things that actually happened to us while building, reported as incidents. How representative they are of anyone else's experience is unmeasured.

Relevant to ROADMAP #2, which currently delegates "how intelligence gets integrated" to the
examples. These are the costs of following them, and none is written down anywhere.

- **Taking `amplifier-agent` broke our test runner.** `amplifier-core` ships a pytest plugin
  that auto-registers on import and then fails wanting `pytest-asyncio`, which we do not
  use. Every test in the repository, including the ones that never touch a model.
  tmux-fleet hit this first and disables the plugin the same way. If this is the paved path,
  the pothole belongs on the map.
- **A credential is not a provider, and a mounted tool is not a loaded tool.** The engine
  resolves providers by *credential presence* and ships **no provider client library at
  all**, so preflight passed and the turn died at mount time with `No module named
  'anthropic'`. With that fixed, `tool-web` failed validation for want of `aiohttp`; the
  engine logged it and **carried on without it**, so the agent had no search and answered
  from memory. Both dependencies are ours to declare, because nothing upstream brings them
  and no extra offers them. The generalisable point: this engine **degrades rather than
  fails**, so an integration must verify the post-condition — what actually mounted — and
  never trust that asking worked.

  *A note on what is NOT a finding here: the install is large (53 packages, 85 MB) and that
  is correct. A smart tool ships its own intelligence; the "thin CLI" in the spec is about
  where logic lives — in the library, not the wrapper — not about dependency footprint.
  Weighing a complex intelligence service against a shell script's install size is a
  category error, and we should not hand the spec's authors that frame. The one specific
  oddity worth reporting is narrower: the engine brings `fastapi`, `uvicorn`, `starlette`
  and `mcp` into a tool that never serves HTTP, plus duplicate `httpx`/`httpx2` and
  `httpcore`/`httpcore2`. That is a remark about `amplifier-agent`'s own dependency graph,
  not about smart tools being heavy.*
- **An undeclared dependency is invisible to CI.** We installed the engine by hand, imported
  it, and declared it nowhere; the job that installs from git was structurally blind to it
  because it installs exactly what is declared. Not a spec issue — a lesson.

*Both agent-integration entries above are install-time evidence. Use-time evidence arrives
when the reasoning turns actually run (M3), and this section should be revisited then.*

---

## What measuring quality taught us, once we could measure it

**Evidence: MEASURED** — live before/after on named fixture sets, every figure from a recorded run in `evaluation/results/`. **N=1 per arm**: run-to-run variance is unmeasured, and a model changes underneath you without telling you.

Everything above was learned by building. This section was learned by **evaluating** — and
it changed our view of what a smart tool owes its caller more than anything else here.

### Asking for a capability is not getting it — four times, one shape

The single most repeated defect in this project, and the one we would most want another
builder to be warned about:

```
provider credential present   →  but no client library installed
tool in the mount plan        →  but never loaded (aiohttp → bs4 → ddgs, found one at a time)
guard checking dependencies   →  checking a hard-coded list that rots
guard reading the mount registry →  reading a registry that does not exist yet
```

The host engine logs a module that fails validation and **carries on without it**. That is a
reasonable thing for a general-purpose agent host to do, and it is catastrophic for a smart
tool, because a research agent with no search **does not fail**. It answers from memory and
returns URLs it never opened. One of our live runs said, in its own words, *"No live access
to the two listed sources"* while two sat in `sources.json` — reporting fabricated evidence
in the voice of caution, and passing every downstream check because a source was *present*.

We wrote the lesson down after the second occurrence and then repeated it twice more,
including in a guard written specifically to fix that class. What finally worked was
**changing the question**: not *"is search configured?"* but *"did search actually happen?"*
— counted from the engine's own tool events. That needs no knowledge of anybody's dependency
list, which is why the first two attempts rotted and this one has not.

**For the spec:** a smart tool that composes a host's capabilities should be expected to
assert its post-conditions on the **outcome**, not the request. We would support a
conformance check along the lines of *"a capability the tool declares must be demonstrably
exercised, or the tool must refuse"*.

### An evaluation that cannot fail is decoration

Both our harnesses have an **adversary mode** that returns deliberately wrong but
*structurally valid* results — valid so they survive the tool's own validators and actually
reach the scorer. It exists purely to prove the scorer bites.

It earned its place immediately: it caught a **false pass in our own harness**, where a
run that failed for mechanical reasons was being scored as an honest refusal. A refusal
because the evidence was inadequate is an answer; a refusal because the machinery broke says
nothing about the question. We had already drawn exactly that distinction inside `fact-check`
and still failed to draw it in the harness we wrote immediately afterwards.

**Per category, never one number.** Our first fixture set scored **14/14 on its first live
pass** — which is not good news. No headroom means nothing can be tuned, because any change
can only hold or regress. A second set whose evidence *contradicts itself* scored 8/12 and
immediately exposed two real failure modes the clean set structurally could not see.

### The worst failure makes your quality signals look better

Asked about two things that **do not exist** — a language and a study we invented — our
research tool gathered **30 and 27 real sources** and returned `confidence: high` for both.
Nothing malfunctioned. Search returned plentiful, genuinely relevant material about adjacent
real subjects, and the synthesis answered as though the subject were real.

Those two runs had *more* sources, valid citations, fluent prose and higher confidence than
the questions the tool got **right**. Every signal a careful reader would check pointed the
wrong way.

This is the one failure mode that cannot be found by reviewing output, however carefully.
Only an input whose correct answer you already know will surface it. **We would urge the
conformance kit, or at least the guidance, to treat "asked about something that does not
exist" as a named test case for any tool that summarises retrieved evidence.**

The fix was one paragraph in the synthesis prompt — *a search returns whatever is closest to
a question, never proof the thing exists* — and it took `nonexistent` from 0/2 to 2/2 with
no regression. The same 30 sources, the opposite conclusion about what they establish. **The
defect was never in retrieval; it was in what the synthesis was willing to assume.**

### Honesty about absence is not free

That fix raised spend **41%** for the same six questions, because saying what the sources
*do* cover takes more words than answering the question as asked. Worth knowing before
anyone budgets for a tool that is expected to hedge well.

And a related measurement, for anyone tempted by an intermediate "sharpen the question"
stage: ours **doubled cost and wall-clock and changed no score** on well-formed questions —
6/6 both ways, $0.63 against $0.31. More sources did not buy a better answer either; the
cheaper arm gathered half as many for one question and reached the same conclusion. We have
not yet tested the vague-question case the stage exists for, and we say so in the log rather
than claiming a verdict we did not earn.

---

## A proposal with evidence: a `skill` verb, and why `--help` is not enough

**Evidence: MEASURED** — a live A/B with two sub-agents, identical task, clean context. **N=1 per arm**, one tool, one model. The direction of the difference is clear; the magnitude (16 vs 8 calls) should not be quoted as a rate.

**This is the finding we would most like the spec to consider adopting**, because unlike the
rest of this document it comes with a measured A/B rather than an argument.

### The observation

`--help` is written for a person. Ours was a good one — 68 lines, verbs grouped, each marked
`(deterministic)` or `MODEL-BACKED`, with sections on using the result and navigating a large
one. It was also, we assumed, sufficient for an agent.

A smart tool's whole premise is that **a caller states an intent and gets a structured result
it can hand straight to code**. If the primary caller is an agent, then the tool's own
description is an agent-facing interface — and we had never tested it as one.

### The experiment

Two sub-agents, **identical task, clean context, neither able to read source or search the
web**. One was given only the `--help` text; the other only a `skill` document rendering the
same tool as an [Agent Skill](https://agentskills.io/specification) — YAML frontmatter, then a
markdown body with a `## Cost` section, a verb table marking each verb deterministic or
model-backed, how to read the result, how to go deeper, and worked examples.

Both were told: work out which verbs are free, **run only those**, price a low-depth run,
explain how to read a large result without swallowing it, and say what `confidence` means.

### The result

| | `--help` | `skill` |
|---|---|---|
| LLM calls to finish | **16** | **8** |
| free vs paid verbs | correct | correct — *"explicit and unambiguous"* |
| cost of a low-depth run | $0.0405 | $0.0405 |
| **what `confidence` means** | **could not answer** | **answered, with a remediation plan** |

Both agents correctly refused to run the paid verb. Both priced the run. The split that
`--help` already marked inline was understood by both.

**The difference was interpretation.** The `--help` agent:

> *"I could not find any mention of a `confidence` field anywhere in the documentation — I
> grepped the doc text directly and it does not appear once… I don't know, and I would not
> fabricate a scale or a remediation policy for a field the documentation never defines."*

The `skill` agent:

> *"One of `low`, `medium`, `high`… the doc says plainly 'when it says low, believe it.' If a
> result came back `low`, I would **not** treat the brief as reliable enough to act on — I'd
> pull `sources <id>` to see what was actually found, and tell the user the question isn't
> well-supported by available evidence rather than passing along a shaky answer as fact."*

### Why this matters more than the call count

**`--help` told an agent how to CALL the tool. The skill told it how to BELIEVE the result.**

An agent that cannot interpret `confidence` will present a low-confidence brief as fact. That
is precisely the failure we spent a day eliminating *inside* the tool — and here it was,
reappearing at the **consumption boundary**, in a tool that had already been fixed.

The reason `--help` omitted it is instructive and not a mistake anyone would catch by
re-reading: a human seeing `"confidence": "low"` in a JSON envelope knows what to do without
being told. **The things a human reader supplies from common sense are exactly the things
that must be written down for an agent.** No amount of care in writing human-facing help
surfaces them, because to the writer they are not missing.

Halving the LLM calls (16 → 8) is the secondary win, and it came from the explicit `## Cost`
section: the agent did not have to spend turns reasoning about which verbs were safe.

### What we would propose

1. **A `skill` verb** — or an agreed flag — on every smart tool, emitting the tool as an
   Agent Skill. Deterministic, credential-free, and cheap to implement: ours is one shared
   renderer plus a per-tool description of its verbs and result contract.
2. **The manifest already carries name, description and requires.** What it does not carry,
   and what an agent most needs, is **how to interpret a result** — what the fields mean and
   what a caller should DO about each value. We would suggest the spec name this explicitly
   as something a tool must document, wherever it chooses to document it.
3. **A conformance check worth considering:** a tool's agent-facing description must define
   every field its result envelope can return. Ours would have failed that check, and the
   failure would have been caught in seconds instead of by a two-agent experiment.

Both agents independently found the same real gap in our own documentation — `classify`'s
flag shape appears in neither document, so both discovered it by trial. That is worth saying
plainly: **the experiment found defects in the thing being tested, which is the only reason to
run an experiment at all.**

---

## A second proposal: let a host answer "can I use this?" BEFORE installing

**Evidence: PROPOSAL — none.** Not implemented and not measured. The *problem* it describes is OBSERVED (our manifest really does declare requirements only in prose) and the caveat about static-versus-verified is MEASURED, in the sense that the failure it warns about actually bit us. The proposed schema itself is an argument, and should be read as one until somebody builds it.

A concrete companion to the `skill` proposal above, and the same shape of problem — the
tool knows something a host needs, and has no machine-readable way to say it.

### The question a host actually asks

*"I have `PERPLEXITY_API_KEY` set. Can I use this Perplexity-powered tool?"*

That is the decision every host, catalog browser and orchestrator makes before touching a
tool. Today a smart tool can only answer it **after** being installed and run.

### What we ship today, and where it stops

`check` answers it well — structured, per-requirement, with provenance:

```json
{"name": "perplexity",   "state": "satisfied", "detail": "resolved from $PERPLEXITY_API_KEY"}
{"name": "ai-provider",  "state": "satisfied", "detail": "$ANTHROPIC_API_KEY, client library present for anthropic"}
```

But the **manifest** — the thing a catalog can read without installing anything — declares
requirements only in prose:

```json
{"name": "ai-provider", "optional": true,
 "purpose": "...Any one of ANTHROPIC_API_KEY, OPENAI_API_KEY, GOOGLE_API_KEY,
             GEMINI_API_KEY or AZURE_OPENAI_API_KEY satisfies it..."}
```

A host cannot evaluate that. It can only read English and guess — at exactly the moment it
most wants certainty, because nothing is installed yet.

### The shape we would propose

Two fields on `requires[]`, both small and closed:

```yaml
requires:
  - name: perplexity
    optional: true
    satisfied_by:
      any_of:
        - {kind: env,  name: PERPLEXITY_API_KEY}
        - {kind: file, path: ~/.config/amplifier-research/credentials.toml, mode: "0600"}
    enables: [research]          # what you LOSE without it
    purpose: "..."               # unchanged, for humans
    install: docs/CONFIGURATION.md
```

and the symmetric half, so the loop closes:

```yaml
capabilities:
  - {name: research, cost: model-backed,  requires: [perplexity, ai-provider]}
  - {name: read,     cost: deterministic, requires: []}
```

A host now computes, **with nothing installed**: evaluate `satisfied_by` against its own
environment, derive exactly which capabilities are available. `kind` needs only a handful of
values — `env`, `file`, `command`, `package` — and `any_of` / `all_of` covered every case we
met.

**`enables` is the load-bearing field, not `satisfied_by`.** "Requirement unsatisfied" is not
actionable. *"You lose `research`; you keep `read`, `sources`, `render` and `list`"* is the
decision a host is actually making, and it is the same information the spec already asks a
tool to put in `purpose` — just in a form something can act on.

### The caveat that must be in the schema, because our own bugs prove it

**A static declaration PREDICTS. Only running `check` VERIFIES.** We have the scar tissue:

```
ANTHROPIC_API_KEY present   →  satisfied_by would say YES
anthropic package absent    →  reality said NO
```

That exact case cost us a debugging session. A `satisfied_by` expressing only `kind: env`
would have told a host "you can use this" and been **confidently wrong**. So:

- `kind: package` must be expressible, and a credential requirement almost always needs
  `all_of: [env-or-file, package]` rather than the env var alone.
- **The spec should name the two states distinctly** — `predicted` (static, from the
  manifest) versus `verified` (from running `check`). A host may *plan* on the first; it must
  not *promise* on it.

This is the same lesson as the "asking for a capability is not getting it" section above,
promoted to schema level. Four defects in this project had that one shape: something declared
present but not actually usable, with nothing noticing. A static `satisfied_by` is that trap
made official — which is an argument for **naming it a prediction in the schema**, not for
leaving it out.

### Two smaller things we would change in our own tool regardless

- **`check` should report `enables` whether or not a requirement is satisfied.** Ours sets
  `lost_without_it: null` when satisfied, which wastes the field — a host wants the gating
  map at all times, not only when something is broken.
- **`credential_surfaces` belongs in the manifest, not only in `check`.** A host that knows a
  tool has a `perplexity` surface can pre-fill it from a credential store it already manages,
  before the tool is ever run.

---

## Measured against the Smart Tool Creator: what we would adopt, and what does not fit

**Evidence: MEASURED** — ran `smart-tool-creator init` and the conformance kit against its
output, on this machine, today. **N=1**, one scaffold, one language target (`uv-python`),
one intelligence target (`copilot-sdk`). The structural comparison against our own tools is
OBSERVED; the judgments about fit are JUDGMENT and marked where they appear.

[DavidKoleczek/amplifier-smart-tool-creator](https://github.com/DavidKoleczek/amplifier-smart-tool-creator)
is a smart tool for building smart tools. We ran it to find out how much of our path it
collapses, and what of ours it is missing.

### What one command produced

```
$ time smart-tool-creator init release-notes --description "..." --skill
Scaffolded release-notes at /tmp/cx1/release-notes
  27 files written as a uv-python tool with copilot-sdk intelligence,
  committed to a new git repository with no remote
  environment synced: `uv run release-notes manifest` works from that directory
  Agent Skill at skills/release-notes/SKILL.md
  reference/ holds shallow clones of amplifier-smart-tools, copilot-sdk, agentskills

real    0m5.230s
```

Conformance kit against that scaffold, **with zero domain code written**:

```
PASS {'pass': 15, 'fail': 0, 'skip': 0}
```

**Our M0 — reaching the same 15/15 — was an entire milestone.** Stated fairly, the targets
differ: we built two distribution roots sharing one library in a monorepo plus CI, and `init`
produces one standalone tool with no remote. But the part that is genuinely like-for-like —
*a conforming skeleton that passes the kit* — went from a milestone to **5.2 seconds**.

A caveat we hit in both directions: the kit reported `10 pass / 5 skip` until the tool was on
`PATH`. The five runtime checks **skip rather than fail** when the binary is not resolvable.
That is the same silent-skip trap we recorded earlier about `manifest-version-matches-package`,
and it bit us twice today — once on our own repo, once on this scaffold. A skip reads like a
pass at a glance.

### Where we independently arrived at the same answer

The strongest signal in this comparison is not a difference — it is a convergence:

```
smart-tool-creator -h        terse summary for a person
smart-tool-creator --help    the tool's skill, written for an agent driving it
```

**That is the design we reached separately** and wrote up above, with an A/B behind it. Two
builders solving it the same way, in the same week, without coordinating, is worth more than
either of us asserting it — and our measurement (16 vs 8 LLM calls; the `--help` agent unable
to say what `confidence` meant) is evidence for a decision he had already made on instinct.

Two more convergences: the skill is a **first-class library capability** (`core/skill.py`),
not CLI text; and the intelligence sits **behind a Protocol** with a `default_intelligence()`
factory — the same split our design note argued for.

### What we would adopt from it, and are missing

| | why |
|---|---|
| **`reference/` shallow clones** | the spec, the SDK and the skills spec sitting in the repo. We fetched these by hand all session |
| **`AGENTS.md` + `CONTRIBUTING.md` from minute zero** | ours were never written; a contributor gets nothing |
| **pre-commit config and a `setup-for-dev.py`** | our dev loop lives in a findings file and my memory |
| **`--help` as the skill** | ours is a `skill` VERB. His is better for discovery: an agent runs `--help` reflexively and will never guess a verb it has not been told about |
| **A committed `uv.lock`** | ours is not committed |

The `--help` one is the most actionable, and it is a **JUDGMENT that our choice was worse**.
We reasoned that a host looking for the skill should find it in the verb list. But a host that
does not know the tool at all reaches for `--help` first, and ours answers that with prose.

### What would NOT work in ours, and why

**1. One intelligence seam, where we found two.** His `Intelligence` Protocol is
`preflight()` plus `run(AgentRequest) -> AgentResult` — an agent runner. Our design note
(above) concluded that evidence *acquisition* and *reasoning* are different seams: Perplexity
is not an agent SDK, it is a research service that owns its own search loop and returns
sources. It has no system prompt, no tool list, and no turn we control. **A single
agent-runner Protocol cannot express it** — you would have to model the whole service as one
opaque `run()` and lose the sources, the per-call accounting and the citation structure that
our entire citation-validation spine depends on.

This is the most useful thing we can hand back: *the interface is right, and one of it is not
enough.* A scaffold that offers `--intelligence copilot-sdk | amplifier-agent` is offering a
choice of **agent SDK**; a research service is a different kind of dependency and wants its
own seam.

**2. Single-root layout.** `init` makes one tool per repository. We deliberately ship two
distribution roots over one shared `research-core`, which is what lets `fact-check
--from-run` read evidence `deep-research` already gathered instead of paying to gather it
again. Adopting the scaffold layout would mean splitting the repo and losing the shared
library, or diverging from the template immediately.

**3. `preflight()` raises; our `check` returns data.** His contract raises an error naming
what to configure. Ours is a deterministic *verb* returning structured per-requirement state
with provenance, which a host can read **without catching an exception** and before deciding
to call anything. JUDGMENT: ours is more useful to a host, his is simpler for the library,
and they are not mutually exclusive — a tool can do both.

### What the creator is missing that we measured

Stated as what a **scaffold default** could carry, since a scaffold settles conventions by
default rather than by argument — whatever `init` emits becomes the norm long before a spec
catches up.

- **The scaffolded `SKILL.md` has `## Install` and `## Use it` and nothing else.** No section
  for what the result MEANS. That is precisely the gap we measured: an agent that can call
  perfectly and misread the answer. A template heading — *"Reading the result: every field
  your envelope can return, and what a caller should do about each value"* — would propagate
  the fix to every tool ever scaffolded.
- **The scaffolded tests cover `manifest` and `skill` only.** Nothing asserts the tool
  verifies its capability actually *ran*. Our four same-shaped defects all had that
  signature, and the check that finally held counts tool events and refuses when there are
  none.
- **Evaluation is a stated goal, not a shipped command** (`manifest` and `init` are what the
  CLI carries today). When it lands, three properties decide whether it is a harness or
  decoration: an **adversary mode**, **per-category** scores rather than one number, and a
  fixture with headroom — ours scored 14/14 on its first pass and could not discriminate.
- **The nonexistent-subject test case.** For any tool that summarises retrieved evidence, we
  would argue this belongs in the scaffold: ask about something that does not exist. It is
  the one failure where every quality signal points the wrong way.

---

## What hypermedia already settled, and what of it survives an AI consumer

**Evidence: OBSERVED** — a live `deep-research` run over 54 sources (`dr-56f6e5ec`, medium
confidence, $0.30681, 5m10s), read through the tool's own navigation. The survey findings are
as well-grounded as its citations; the judgments about what transfers to an agent consumer are
**JUDGMENT**, and marked.

Our tools return a brief plus a pointer, with a `next` block of follow-up commands. We invented
that. The problem it solves — a client discovering how to traverse a resource it cannot hold —
is one the web spent twenty years on, so we asked what was already settled. **We ran the
question through our own tool**, which is the least we owe a research tool.

### What the formats actually do

| | how a client learns what it may do next |
|---|---|
| **HAL** | `_links` only: relation → Link Object. **No action construct**; method and body semantics live out-of-band in documentation or a framework |
| **JSON:API** | navigational `links` only (self, related, pagination). No action object — and their own issue tracker shows this debated as a **known, unresolved gap**, not an oversight |
| **Siren** | **the one format that formally separates the two**: `links` (rel + href) and `actions` (parameterised, with fields) |
| **Hydra** | RDF `operation` objects |
| **Collection+JSON** | query/template objects carrying parameters |

Two answers we expected to be settled and are not: **no format mandates IANA-registered
relations**, and for a result whose total size is unknown until traversed, all of them simply
emit a next-link until it is absent, **with no required count**.

### The eight failure modes practitioners actually report

Ignored controls and hardcoded paths · generic hypermedia clients never materialising · the
relation vocabulary becoming an undocumented coupling point · weak tooling and code generation ·
payload bloat and extra round trips · affordances going stale between generation and invocation ·
teams cutting back to a minimal subset · and `204 No Content` **breaking the interaction loop,
because a response with no representation carries no links**.

### The inversion, and it is the finding

**Half of that list is about a consumer we are not.**

Failure modes 1, 2 and 4 all describe *human developers writing compiled clients*: they read the
docs and hardcode endpoints, nobody builds the generic client, and the tooling favours fixed
contracts. **An agent consumer inverts every one of them.** It re-reads the affordances on every
call, has no compiled client to regenerate, and cannot hardcode a path it was never told. The
central historical criticism of HATEOAS — *the generic client never showed up* — reads very
differently when **the generic client is exactly what an agent is.**

But two transfer, and hard:

- **The vocabulary is an undocumented coupling point.** An agent still has to know what
  `read_report` *means*. Affordances in the response are necessary and not sufficient; their
  semantics have to live somewhere the agent reads. That is precisely the job of the skill
  document — which we can now say is not a nice-to-have but **the half of the design hypermedia
  never solved**.
- **`204 No Content` breaks the loop.** Directly applicable: **our refusals must still carry
  navigation.** A run that fails with `NoEvidence` today returns an error envelope and no way
  onward, which is the same defect in a different costume.

And one is a validated target rather than a warning: teams that kept hypermedia **kept `self`,
pagination, and a small set of state-dependent actions** and dropped the rest. That is roughly
the shape we arrived at independently, and it is the shape that survived contact with
production.

### What we are taking

**Siren's separation of links from parameterised actions**, because our `next` entries are
commands with arguments, not URLs — they were always actions, and we had no name for that.
**Nothing from the RDF or registered-vocabulary direction**: rejected, because the coupling it
removes is one an agent that reads a fresh skill on every call does not have, and it buys that
at a cost in payload and tooling the practitioners were right about.

### A note on the run itself

Mid-run, the tool **rejected one of its own claims**: *"no source in the provided list directly
supports it as an independent, attributable finding; it is unsupported by the given evidence and
is not asserted here."* Our citation validation caught a fabrication inside our own design
research, on a topic where a plausible-sounding generalisation would have passed any human read.
That is the first time the discipline paid for itself on something other than a fixture.

---

## Building the pattern: what we learned implementing it on ourselves

**Evidence: MEASURED** for the figures below — live evaluation runs, a live A/B, and a real
process killed mid-flight, each cited with its numbers. **N=1 per arm** throughout. The
generalisations after each result are **JUDGMENT**.

We proposed a `parts` vocabulary to the spec (`proposals/output-is-bigger-than-the-response.md`)
and then implemented it in our own tools. Building it taught us more than designing it did.

### A refusal is a response, and ours were dead ends

The sharpest transferable lesson from the hypermedia survey was `204 No Content`: a response
carrying no representation carries no way onward. **Our refusals had exactly that defect** — a
`NoEvidence` returned an error envelope and stranded the caller.

Now every refusal carries typed affordances. Verified with no model and no credential:

```
refused: no_evidence
  check    $0.00  deep-research check              / deep_research.check()
  status   $0.00  deep-research status dr-e1f19ba2 / deep_research.status(...)
  list     $0.00  deep-research list               / deep_research.list_runs()
```

Two constraints that only became obvious once we built it. **Every affordance offered on a
refusal must be free and credential-free** — a caller that has just been refused may be on a
host with nothing configured, and offering it something it cannot run is offering it nothing.
And **both invocation forms must be carried**, because the library is the tool: a library
caller reading a shell command would have to shell out to follow its own tool's advice.

### The generality test passed by being boring

A vocabulary invented for research reports that needs special-casing to describe a thumbnail is
research plumbing with a general name on it. So we expressed an image response with the same
types — `thumbnail` / `web` / `original` at 18KB, 240KB and 21MB — and asserted the thing the
whole design exists for: **an agent can decide it does not want 21MB without fetching 21MB.**

It passed on the first run and **that is the result**. Not that it works, but that no type
needed changing. Had it wanted one `if media_type ==` branch, we would have known.

### Size and time are one problem, and the proof is a `kill -9`

`--detach` returns **part one in 0.1 seconds**: the run id, where the rest appears, and an
explicit `not_yet_true` list, because an accepted request looks a great deal like an answer if
nobody says otherwise.

Rejoining a real 120-second run:

```
t+20s   growing   stage=scope       0/4  poll_in=10
t+80s   growing   stage=synthesise  2/4  poll_in=10
t+120s  final     stage=None        4/4  poll_in=None
```

Then we detached a run and killed the child mid-gather:

```
record status : running     <-- the stale state that would strand a caller forever
LIVENESS      : abandoned
why           : the run record says 'running' but process 325830 is gone, so nothing
                is going to finish it. Whatever reached disk is all there will be.
```

**The record still says `running`, and always will.** A process that dies never writes its own
epitaph. Every status field in a system like this is written by something that has to survive
to write it — which means the one state you most need to detect is the one state nothing can
report. Liveness had to come from outside the record entirely.

**This is the part of the proposal we would defend hardest.** Without it, detaching is strictly
worse than blocking: a caller polls forever for a result that is never coming. PID reuse is a
known hole and we say so rather than implying more certainty than we have.

### The skill measurement, repeated — and what it found this time

An agent given **only** the skill, and a 78-line report over 384 lines of sources:

```
status dr-56f6e5ec      1 line
read --lines 40         partial: "38 lines sit beyond this window"
read --lines 78         complete
```

It called `status` first as taught, treated the completeness block as an instruction — *"I
treated that as an explicit instruction not to answer from the partial slice"* — and **never
touched `sources.json` at all.**

But it also found a real defect, and the defect's shape is the finding. Our completeness note
says *"ask for specific `--sections`"*. The flag **exists and works**. The skill never mentioned
it. So the agent followed our advice exactly as written and could not comply.

**Nothing was broken. No test could have failed.** The defect lived entirely in the gap between
two documents that were each correct on their own. No amount of re-reading either would have
surfaced it — only putting an agent in front of them with a real task did.

### A run directory is a public surface, so name it like one

A second tool reads ours via `--from-run`, and so does any agent handed a `path`. Three rules:
**the extension says HOW to read it** (`.json` one document, `.jsonl` append-only lines, `.md`
prose, `.log` unstructured with no schema promised); **the name says WHAT it is** (singular for
the thing, plural for a collection); **a subdirectory says WHOSE it is** (`raw/` holds somebody
else's bytes, verbatim).

Two consequences we hold to: **`.log` is the only file with no schema**, so anything code must
read never gets that extension; and **`run.json` is the only file rewritten**, so a reader
racing a writer sees a whole earlier version rather than half of a newer one.

Writing the rule down exposed three artifacts — `scope.json`, `attempts.json`, `detached.log` —
that existed and were documented nowhere.

### The evaluation, since a tool nobody measured is a tool nobody should trust

All four modes, both tools:

```
                 oracle        adversary     live
fact-check       14/14 = 1.0   0/14 = 0.0    14/14 = 1.0   $0.053388   129.9s
deep-research      6/6 = 1.0    0/6 = 0.0      6/6 = 1.0   $0.654096   601.5s
```

**Adversary mode is the one that matters** — deliberately wrong but structurally valid answers.
A harness still scoring well there measures nothing. Both drop to exactly zero.

Per category, never one number, and two integrity counters scored separately because an
aggregate would bury them: **dangling citations: 0** (no marker pointed at a source the run did
not have) and **asserted falsehood from absence: 0** (not once did "no evidence for X" become
"X is false").

The row carrying the most weight is `nonexistent 2/2`. **It was 0/2 before the hallucination
fix.** A green row there is not "the tool works" — it is "the defect we introduced is still
gone". The fixtures worth keeping are the ones that once failed; a suite of cases that never
caught anything measures its author, not the tool.

---

## Open question: who does the chaining?

**Evidence: OBSERVED** for what our tools do and do not support today, verified by running
them. **PROPOSAL** for everything after that — not implemented, not measured. Raised as food
for thought rather than a recommendation.

Right now, when an agent wants `deep-research` and then `fact-check`, **the harness is the
thing doing the chaining.** It makes one tool call, receives a result, holds it in context,
decides what to extract, and makes a second call. Two round trips, and the intermediate data
sits in the agent's context whether or not it is ever read again.

That is the same context problem the `parts` proposal addresses, one level up. We spent this
project making a single result navigable without flooding a caller. **Composition is that
problem across tools instead of within one.**

### What already works, and it is the interesting half

```bash
fact-check check-claims --from-run dr-56f6e5ec --claim "..."
```

`--from-run` lets one tool consume another's evidence **without the harness carrying it**. The
run directory is the hand-off, so 384 lines of sources move between two tools and never enter
the agent's context at all. That was built for cost — not re-paying for evidence — and the
composition property was a side effect we did not design for.

The general shape worth noticing: **a shared, durable, addressable substrate is what lets two
tools compose without a broker.** Not a pipe, not an API — a place both agree to look.

### What does not work, and the gap is specific

We checked. **Nothing in either tool reads stdin.** Every verb takes flags only. So chaining
two of our CLIs today means surgery on the envelope:

```bash
deep-research research --query "..." \
  | jq -r .result.run_id \
  | xargs -I{} fact-check check-claims --from-run {} --claim "..."
```

That works, and it is not something an agent should have to invent. The `run_id` is buried in
a JSON envelope, so the pipe needs a JSON tool in the middle, and the caller must already know
that `--from-run` is the flag that accepts it.

### The two forms, and neither should be second-class

**Library composition** works today and is the cleaner of the two — `import deep_research,
fact_check`, call one, pass what you want to the other. Anyone embedding these in an
application already has it.

**CLI composition** is where the value is for a harness, precisely because a single `bash` call
running a pipeline is **one tool call instead of N**, with the intermediate data never touching
the agent's context. That is a large win and we do not support it properly.

### The answer is probably Unix, not a protocol

Our first instinct was to design a convention — a handshake, an envelope contract, affordance
commands declared composable. On reflection that is the wrong shape. **Unix solved this, and
anything we invent will be worse than what already works.**

A pipe carries bytes. A tool reads stdin when it is given stdin. That is the whole mechanism,
it is forty years old, and every harness on earth can already drive it:

```bash
deep-research research --query "..." | fact-check check-claims --claim "..."
```

Nothing in that line needs a new protocol. It needs one thing we do not currently do: **read
stdin when there is something on it.** We checked — neither tool does. That is the gap, and it
is a small one.

The reason this matters for a harness is arithmetic. A pipeline is **one tool call instead of
N**, and the intermediate data never touches the agent's context. Not because we designed a
clever hand-off, but because that is what a pipe has always done.

### Where a pipe genuinely cannot reach, and only there

Two cases where piping the content is the wrong move, and a **handle** is the right one:

- **The output is not finished.** You cannot pipe a result that does not exist yet. A detached
  run returns in 0.1 seconds with an id and nothing else worth piping.
- **The output is too large to be worth moving**, or the downstream tool needs only part of it.
  Our 384-line source list is a poor thing to push through a pipe so the next tool can read
  three entries.

For those, pass the **run id** and let the downstream tool resolve it against the shared runs
directory — which is exactly what `--from-run` already does, and which worked before we thought
of it as composition.

This is not a new idea either. **Unix does the same thing**: you pipe data when data is the
right thing to move, and you pass a *filename* when it is not. A run id is a filename with a
tool-shaped name on it.

```
small, complete, synchronous   →  pipe the content          ordinary Unix
large, partial, or in flight   →  pipe the handle           --from-run <id>
```

The second is the only part that needs a convention at all, and the convention is one sentence:
*a smart tool that produces durable output should give it an id, and accept that id from
another tool.*

### Our own tools do not need this, and that is the useful part

We were about to wire stdin. We are not going to, and the reason is worth more than the feature
would have been.

**Our two tools already compose, and a pipe would add nothing.** They share a runs directory,
their outputs are large and durable, and `--from-run` moves a 12KB source list by reference
instead of by value. Piping the content would be strictly worse. So for this pair, the honest
answer is that handle composition is not a fallback for the async case — it is simply the right
mechanism, and the pipe is the thing we do not need.

That inverts the question usefully. Rather than *should smart tools read stdin*, ask **what
makes a tool need a pipe at all**:

| a tool composes by HANDLE when | a tool needs a PIPE when |
|---|---|
| it writes durable output somewhere both parties can reach | it produces a value, not an artifact |
| its output is large, or partial, or not finished yet | its output is small and complete |
| it shares a substrate with the next tool — same runs directory, same store | the next tool is **somebody else's**, with no shared store |
| it is one of a family built together | it is a stateless transform: classify, translate, extract, reshape |

**The shared substrate is what removes the need for a pipe.** Two tools that agree on a place
to look do not need to move bytes between them. Two tools that do not — different authors,
different machines, nothing in common but a terminal — have the pipe and nothing else.

Our tools are the first column. A great many smart tools will be the second, and **those are the
ones the spec should be thinking about**, because they are the ones with no other option. A
stateless classifier that cannot be piped into is a tool that can only ever be driven by a
harness, one call at a time, with every intermediate landing in an agent's context.

### The question for the spec

Narrower than a protocol, and in two parts:

1. **Should a smart tool whose output is a value be able to read stdin?** Our position, held
   lightly: yes, and via ordinary Unix rather than anything designed. A tool that reads stdin
   when given stdin can be composed by every harness that already exists.
2. **Should a tool whose output is durable expose a handle for it?** Yes — and that is the part
   we *have* built and can point at, because it is what lets a 384-line source list move between
   two tools without entering anyone's context.

Neither needs a protocol. The failure mode of designing one here is that each tool ends up
needing to understand every other tool's envelope, which is coupling by another name.

**What we can and cannot claim.** We have one working instance of handle composition
(`--from-run` over a shared runs directory) and **zero** of pipe composition. We are not
building the second, because our tools genuinely do not need it — and a feature we would not
use is a feature we could not honestly evaluate. The observation stands on the reasoning above,
not on a measurement, and it is offered as food for thought rather than a recommendation.

---

## The `-h` / `--help` split is a gesture, and a gesture has to hold everywhere

**Evidence: MEASURED** for the original A/B (16 vs 8 LLM calls) and **OBSERVED** for
everything after — each defect below was found by running the tool, and every line count is
real output.

We adopted the split from the Smart Tool Creator after measuring our own choice and finding it
worse: **`-h` is the terse table for a person, `--help` is the document for an agent.** Then we
applied it to the root command and stopped.

`deep-research research -h` and `deep-research research --help` were **identical** — 35 lines
of argparse table. An agent drilling into a specific verb, which is the natural move when
deciding *how* to call something, got a flag reference instead of an explanation.

### This is the same defect three times

| what happened | how it was found |
|---|---|
| Our completeness note told callers to use `--sections`; the skill never mentioned it | an agent followed our advice and **could not comply** |
| `--no-scope` shipped with carefully measured help text; invisible in the skill | asked "is this clear in the skill?" and checked |
| Every subcommand answered an agent with an argparse table | asked "does this hold for subcommands?" and checked |

Each time a capability was **present in the tool and undiscoverable by its consumer**. Each
time a green test suite. The test written after the *first* occurrence was called
`test_the_skill_documents_every_flag_our_own_messages_advertise` and asserted that one flag
appeared — **it passed on the commit that introduced the second occurrence.** A test named for
a class that checks an instance is worse than no test: it occupies the slot where the real
check would go.

### What we now hold to

**The split is a gesture, not a feature of the root command.** `-h` always means *terse, for a
person who already knows this*. `--help` always means *the agent-facing document for whatever
scope you asked about*. At the root that scope is the tool; at a verb it is that verb — what it
does, **whether it spends money**, every flag and what it is *for*, and how to read the result.

Every verb now answers differently:

```
manifest  -h=9   --help=14      research  -h=37  --help=31
config    -h=17  --help=20      check     -h=14  --help=18
status    -h=13  --help=18      classify  -h=10  --help=18
```

**And the root skill must say the deeper documents exist.** An overview that does not mention
them hides them — which is the same defect once more. Ours now tells an agent: *this document
covers the tool; every verb has its own, ask `<verb> --help` when you are about to call
something.*

### Two implementation notes that are the difference between a fix and a fuse

**Wire it as a post-pass, not at each call site.** Ours walks the subparser set, removes
argparse's single `-h/--help` action and installs two that differ. Our verbs are created in two
places — some per-tool, some in a shared package — and more importantly **a verb added next
year inherits the behaviour without anyone remembering.** A fix that depends on the next author
remembering is the same defect with a longer fuse.

**Enumerate in the test; never spot-check.** Ours walks every verb of every tool and asserts
`-h` starts with `usage:` while `--help` starts with `---`. Naming one case is precisely how
the second and third occurrences got through.

---

## Two harnesses: what a second host told us that the first could not

**Evidence: MEASURED** — four agent runs plus one real paid run, each with its commands and
numbers recorded. Claude Code 2.1.263 and Amplifier, same tool, same task.

Every agent measurement in this project had used **one host**: Amplifier, our own harness, our
own conventions, our own idea of what a skill is. That cannot distinguish *"we built something
portable"* from *"we built something Amplifier-shaped."* So we ran it in a host we did not
design for.

### The four arms, and the wrong answer came from us

The task: a sharp question, a tight budget, a nearly-full context. The correct call is
`--depth low --no-scope --no-inline`.

| host | documentation | steps | final command | |
|---|---|---|---|---|
| Amplifier | old skill + `-h` | 5 | `--depth low --no-scope --no-inline --quiet` | correct |
| Amplifier | new skill, **bad rule** | 9 | `--depth low --no-inline` | **wrong** |
| Amplifier | corrected skill | 7 | `--depth low --no-scope --no-inline` | correct |
| Claude Code | corrected skill | 6 | `--depth low --no-scope --no-inline` | correct |

**The only wrong answer in four arms came from our own documentation, not from any host.**

We had measured that the scope stage does not pay on questions that are already sharp. We then
wrote the skill as *"pass it when a program composed the question, leave it off when a person
phrased it"* — turning a property of the **question** into a property of its **author**. A
person can ask a perfectly sharp question, which is exactly what the task was.

The arm reading that text refused the saving and said why:

> *"This question was phrased by a person, not composed programmatically, so the doc's own rule
> says leave it off — regardless of how well-bounded the question looks. I did not try to
> override that judgment call with my own assessment of clarity, since the doc gives an explicit
> provenance-based rule, not a clarity-based one."*

It left ~27% of the cost on the table by following our instruction **faithfully**. The arm with
the older, scrappier docs noticed the two texts disagreed, judged the more precise one
authoritative, and got it right.

**A better-presented document is obeyed more exactly. That is a multiplier, not an improvement**
— it amplifies a wrong rule as faithfully as a right one. The lesson we would pass on: gate
presentation work on the text being *verified*, not the reverse.

### A number is easy to carry in the wrong direction

Fixing that exposed a second one of the same kind. Our prose said `--no-scope` *"saves ~37%"*.
It does not. **37% is what scope ADDS; skipping it SAVES ~27%.** One measurement, two numbers,
and every place we had written it quoted the flattering one. Wall-clock had the identical error:
+69% with, −41% without.

Twice in two days our prose outran the measurement, and **both times an agent trying to ACT on
the text found it — never a human re-reading it.** A ratio changes meaning with its denominator,
and the sentence reads fine either way, which is precisely why it survives review.

### `--help` and the installed skill are no longer the same document, on purpose

We had a hard invariant: `--help` byte-identical to `SKILL.md`, so drift was impossible. Then we
noticed — by reading David's generator, not our own tool — that **every skill his scaffold emits
carries an `## Install` section and ours carried none.**

That matters more than it looks. **`npx skills add` installs a DOCUMENT, not the program.** A
host that gains the skill without the binary holds a description of a command it cannot run, and
nothing in the document tells it how to fix that. Our manifest carries `requires[].install` —
reachable only by running the tool, which is the thing it cannot do.

But adding it to both made `--help` incoherent: **its reader already has the binary.** Two
readers, two needs.

The resolution was to notice what the invariant actually was. It was never *"these two strings
are equal"* — it was ***"one source, and the second artifact is mechanically derived from it"***,
with equality being the cheapest derivation that happened to work until now.

```
--help      136 lines   pure usage, zero acquisition instructions
SKILL.md    149 lines   the same document, install block spliced in
test        SKILL.md == compose_skill_file(--help)
```

Still impossible to drift. Each reader gets the document that is true for them.

**And the first version of that was broken in a way no test caught.** We *prepended* the install
block, which pushed the YAML frontmatter from line 1 to line 14 — and frontmatter is how a host
learns a skill's name and description, which is to say how it discovers the skill at all. We had
silently broken the one thing the file exists to do. Caught by looking at the output. **When you
change a file's shape rather than its content, every check guarding its content still passes.**

### Installing and running it from Claude Code

```
npx skills add colombod/amplifier-smart-tools-research --agent claude-code -y
  ✓ Found 2 skills
  ✓ deep-research → ./.claude/skills/deep-research
  ✓ fact-check    → ./.claude/skills/fact-check
  byte-identical to what we generate · frontmatter line 1 · install block line 11
```

The install command our skill documents works from cold:

```bash
uv tool install 'git+https://github.com/colombod/amplifier-smart-tools-research#subdirectory=tools/deep-research'
→ Installed 1 executable: deep-research
```

Then a **real run, real money**, driven entirely from the installed skill:

```
run dr-aa7a2ff0   $0.05799 actual   12,597/3,893 tokens   15 sources   confidence medium
flags chosen by the agent: --depth low --no-scope --no-inline
```

It returned a correct, substantive answer. **And unprompted, it reported which claims were thinly
sourced** — *"only two sources were explicitly pinned to the claims… The join-semilattice framing
had no source pinned to it in the gathered evidence."*

That is the result we would point at above all the others. **The citation-integrity discipline
survived into a foreign harness and reached the end user**, rather than being smoothed into a
confident summary by the host in between.

It also kept the report out of its own context and closed by naming the run directory and
`read <id> --lines N` "at no further cost" — the navigation ladder used correctly by a host that
had never seen it before.

### Every gesture transferred

`-h` versus `--help`, per-verb documents, the ladder, affordances, `liveness.state`, the install
block. **Nothing we invented failed to survive the host change.** For a set of conventions
designed against a single harness, that is a better result than we expected, and it is the part
we would most like someone to try to break.

### What the real run cost us in credibility, and what we changed

**The estimate was 2× low.** `estimate --depth low --no-scope` said $0.0296; the run billed
$0.05799 — the `depth=low` profile assumes 8 sources and the run gathered 15. Claude Code noticed
and explained it without being asked.

We tell callers to estimate before deciding. A caller that budgets on $0.03 and is billed $0.06
has been misled by the verb whose entire job is preventing that. Open, filed, not yet fixed.

Two changes we did make, both prompted by agents hitting a wall:

- **`estimate` can now price the decision.** It rejected `--no-scope` outright, so a caller doing
  exactly what we told it to do got a number that could not reflect the choice it was about to
  make. It now applies the measured ratio and names its source in the `basis` field.
- **`--max-sources` is explicitly NOT modelled**, and `estimate --help` says so and why. We have
  no measured cost-per-source, and an estimator that silently accepted the flag while ignoring it
  would be worse than one that rejects it. Claude Code read that and declined to guess — which is
  exactly what the sentence was written to cause.

### The honest gap

Three probes, three real defects, and **none was findable by reading**. Reviewing your own docs
tests whether they are consistent with what you meant. Running an agent through them tests
whether they are sufficient for someone who was not there. **Only the second is the actual
requirement**, and it is cheap: each of these runs cost cents and minutes.

---

## What we intend to feed back

**Evidence: SUMMARY** — a routing list, not a claim. Each item's evidence is whatever its own section carries.

| ROADMAP item | What we will have to offer |
|---|---|
| #1 long-running and expensive calls | a tool that genuinely runs for minutes, with progress both streamed and persisted, and a detached mode designed but backlogged |
| #2 integrating intelligence | a copyable seam: a protocol, two implementations behind it, engine imports deferred, and a test strategy that exercises every model-backed path with no credentials |
| #4 AI provider interface | a second data point — a *domain* backend seam, not a chat provider seam — and what it takes to keep two of them honest |
| unnamed: large results | the artifact-plus-pointer convention, and navigation hints carried in the result |
| #5 continuing a smart call | evidence either way. DTU's model calls are single-turn-per-boot with no session continuation, which is evidence the need is smaller than it looks; our staged runs will test that directly |

#3 (host consumption) and #6 (generated wrappers) are out of our scope unless something
falls out for free.

### Proposals aimed at the conformance kit rather than the prose

These are the ones with a concrete shape, ordered by how cheap they look to add:

| Proposal | Why | Cost |
|---|---|---|
| `install` must point at a document that exists | it is checked not to be a command; a dangling pointer is handed to someone already stuck | one file-exists check per entry |
| a tool must be able to report whether its own prerequisites are present | the spec says detection is the tool's job and then never checks it happens | needs a convention for the verb first |
| every CLI verb has a library equivalent | the rule is called absolute, nothing checks it, and the spec's own fixture does not demonstrate it | hard to check generically; publishing the test pattern may be the realistic version |
| a tool declares its configuration surface in `SMART_TOOL.md` | the spec says nothing about configuration lookup and the two reference tools disagree completely — one has a config file, the other has none, and neither declares it | a manifest field plus a presence check; the harder half is agreeing the convention first |
| a statement on whether a tool may read its host's configuration | a smart tool that reads `~/.amplifier/settings.yaml` silently becomes an Amplifier tool, which contradicts "consumable anywhere" | prose, not a check — but it decides whether piggybacking is a feature or a defect |

### How we deliver this

Upstream has **issues, discussions and pull requests all disabled** — the repo says it is
not currently accepting external contributions and invites forking instead. So there is no
inbox to file against, and this reaches the spec's authors through a person rather than a
repository. The team is being asked how it wants findings shared; until then this document
is the artifact and it stays here.

**It lives in a workspace with no remote.** That is a known risk, accepted for now, and the
reason to settle the channel before teardown rather than after.
