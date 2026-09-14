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
