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
---

## What we read

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

We think **automatic** piggybacking is wrong and that the spec should say so:

- A smart tool is host-agnostic by definition — *"consumable anywhere: Copilot, Claude Code,
  a Python service, a shell script"*. Reading one host's private config file makes it that
  host's tool.
- Silently inheriting a credential from another application's file is a security surprise.
  The user configured that key for Amplifier, not for whatever Amplifier happened to invoke.
- It is a private format belonging to another project, free to change without notice.

**Explicit opt-in** is the defensible middle: a setting a user deliberately turns on that
says "also look in this host's configuration". It removes the double-configuration
annoyance without making the tool secretly host-coupled.

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

---

## Friction on the paved path

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

## What we intend to feed back

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
