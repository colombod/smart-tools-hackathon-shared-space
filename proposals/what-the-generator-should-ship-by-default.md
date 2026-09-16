# Proposal: what a smart tool generator should ship by default

**To:** [DavidKoleczek/amplifier-smart-tool-creator](https://github.com/DavidKoleczek/amplifier-smart-tool-creator)
**From:** colombod

**Evidence: MEASURED** for every defect cited — each is something we shipped, or an agent
failed on, with the run behind it. **PROPOSAL** for every suggested default: none of it is
implemented in the generator, and the classification is our judgment.

We built two smart tools and kept a list of everything that bit us. This sorts that list by a
single question: **can a generator make this structural, or is it only advice?**

That question matters more than the individual lessons. Whatever `init` emits becomes the norm
for every tool built from it, long before a spec catches up. **A lesson in a scaffold
propagates. A lesson in a findings document does not.**

## The sort

| what we learned | can a generator do it? |
|---|---|
| A result needs a "what this MEANS" section in its skill | **template default** |
| A test must prove the capability actually *ran* | **generated test** |
| A refusal must carry a way onward | **generated type** + conformance check |
| Readiness should be readable as data, not only raised | **generated interface** |
| Unreported cost and zero cost are different values | **generated type** |
| Evidence-summarising tools need a nonexistent-subject case | **generated fixture**, conditional |
| An eval harness needs an adversary mode | **template default** |
| Anything detachable needs growing / final / **dead** | **template default**, conditional |
| The skill must document every flag the tool's own messages name | **generated test** |
| A run directory is a public surface and wants a naming rule | **advice**, mostly |

Eight of ten are automatable. That is the finding: **most of what we learned the hard way is
not wisdom, it is a missing default.**

---

## The four we would ship first

### 1. A `SKILL.md` heading for what the result MEANS

The scaffolded template has `## Install` and `## Use it`. Nothing about how to read what comes
back.

We measured the cost of that gap. Two agents, identical task, clean context — one given prose
help, one given a skill that explained the result fields: **16 LLM calls versus 8**, and the
prose-fed agent **could not say what a `confidence` field meant** while the other quoted the
rule and planned around a low value.

A heading in the template — *"Reading the result: every field your envelope can return, and
what a caller should do about each value"* — propagates that fix to every tool ever scaffolded,
at the cost of one line.

### 1b. The `-h` / `--help` split, wired at every level and enforced by enumeration

The creator already ships the split at the root, and that is the design we adopted after
measuring our own worse choice. What a scaffold could add is the part we got wrong: **make it a
gesture rather than a feature of the root command.**

`-h` always means terse-for-a-person. `--help` always means the agent-facing document for
whatever scope was asked about — the tool at the root, **that verb at a subcommand**. Ours
answered an agent with an argparse table at every verb until we checked.

Three things worth generating, and the second two are what make it stick:

- **A per-verb document**, rendered from the subparser the author already wrote: what the verb
  does, whether it **spends money**, each flag and what it is *for*, how to read the result.
- **Wiring as a post-pass** over the subparser set, so a verb added later inherits it. A fix at
  each call site depends on the next author remembering, which is the same defect with a longer
  fuse.
- **A generated test that enumerates**: every verb's `-h` and `--help` must differ, and the
  root skill must point at per-verb `--help`. We had a test for this class that checked a
  single instance, and it passed green on the commit that introduced the next occurrence.

**An overview that does not mention the deeper documents hides them** — the root skill should
tell an agent to ask a verb directly.

### 2. A generated test that the capability actually RAN

**Our most repeated defect, four separate times**: something was *declared* present and was
never actually usable, and nothing noticed.

```
provider credential present      →  but no client library installed
tool in the mount plan           →  but never loaded
guard checking dependencies      →  checking a list that rots
guard reading the mount registry →  reading one that does not exist yet
```

The scaffolded tests cover `manifest` and `skill`. Neither asks the question that would have
caught any of these: **did the work happen?** A model-backed capability whose tools silently
failed does not fail — it answers fluently from memory and returns plausible nonsense.

The check that finally held for us counts the engine's own tool events and refuses when there
are none. A generated equivalent — *assert this capability produced evidence of doing
something* — would inoculate every scaffolded tool against the failure mode that cost us most.

### 3. An error type that cannot be a dead end

Hypermedia learned this with `204 No Content`: a response carrying no representation carries no
way onward. **Ours had the same defect** — a refusal returned an error envelope and stranded
the caller.

A generated error type that takes affordances as a **required** argument makes this hard to get
wrong by construction:

```python
raise NoEvidence(message, remedy, affordances=[...])   # the third is not optional
```

Two constraints we only found by building it: **every affordance offered on a refusal must be
free and need no credential** (a caller that was just refused may be on a host with nothing
configured — offering it something it cannot run is offering it nothing), and **both a CLI form
and a library form must travel**, because a library caller reading a shell command would have
to shell out to follow its own tool's advice.

And the conformance check that would have caught us: **every response a tool can return,
including errors, carries at least one way onward.**

### 4. An evaluation harness with an adversary mode

Evaluation is a stated goal of the creator, not yet a shipped command. When it lands, one
property decides whether it is a harness or decoration.

Ours has three modes, and the middle one is the one that matters:

```
                 oracle        adversary     live
fact-check       14/14 = 1.0   0/14 = 0.0    14/14 = 1.0
deep-research      6/6 = 1.0    0/6 = 0.0      6/6 = 1.0
```

**Adversary feeds deliberately wrong but structurally valid answers.** A harness that still
scores well there is measuring nothing, and you cannot tell from a green oracle run. It is free
to run, it needs no credentials, and it is the only mode that proves the scorer can fail.

Two more properties worth generating: **per-category scores, never one number** — an aggregate
hides the only thing worth knowing, since most cases in any realistic set are easy — and a
fixture with **headroom**, because ours scored 14/14 on its first pass and could not
discriminate anything until we made it harder.

---

## Three smaller ones, cheap to generate

**Readiness as data.** `preflight()` raising is right for a library guard; a host also wants to
*read* readiness before deciding to call anything. Ours returns structured per-requirement
state with provenance:

```json
{"name": "ai-provider", "state": "satisfied",
 "detail": "$ANTHROPIC_API_KEY, client library present for anthropic"}
```

Note that line: **credential present and client library present are different facts**, and we
learned the distinction by shipping a tool that had the first and not the second.

**Unreported cost is not zero cost.** We shipped exactly this bug: read a flat `cost_usd` that
did not exist on the response shape, recorded `null`, and every call went invisible in our
accounting while the caller was genuinely billed. A generated type where the field is
`str | None` with `None` documented as *unreported, never free* makes the omission visible.

**A skill must document every flag the tool's own messages advertise.** Our completeness note
told a caller to use `--sections`. The flag existed and worked. The skill never mentioned it.
An agent followed our advice exactly as written and could not comply — and **no test could have
failed**, because each document was individually correct. A generated check comparing flags
named in messages against flags documented in the skill is mechanical.

---

## The one that is conditional, and the reason matters

**Anything detachable needs `growing` / `final` / `dead`.**

We detached a real run and killed the child mid-flight:

```
record status : running     <-- the stale state that would strand a caller forever
LIVENESS      : abandoned
why           : the run record says 'running' but process 325830 is gone, so nothing
                is going to finish it. Whatever reached disk is all there will be.
```

**The record still says `running`, and always will.** A process that dies never writes its own
epitaph — every status field is written by something that must survive to write it, so the one
state you most need to detect is the one nothing can report.

Without that distinction async is **strictly worse than blocking**: a caller polls forever for a
result that is never coming.

We are *not* proposing a flag name. We have one implementation, which is nowhere near enough to
standardise a spelling from. The obligation is that a caller can learn whether work is still
happening; how a tool spells it is its own business.

---

## What only advice can do

**Naming inside an artifact directory.** Ours settled on three rules — the extension says *how*
to read it, the name says *what* it is, a subdirectory says *whose* it is — and two hard
consequences: `.log` is the only file with no schema, and only one file is ever rewritten, so a
reader racing a writer sees a whole earlier version rather than half a newer one.

A generator cannot know what artifacts a tool will produce. But it could emit the rule as a
comment in whatever directory-writing code it scaffolds, which is where an author will be when
the question arises.

## What we have not earned

None of this is implemented in the generator, and we have not tried to implement it there. Our
evidence is one project, two tools, one language. Several items are conditional on a tool's
shape and would be wrong applied universally — the nonexistent-subject fixture means nothing for
a tool that does not retrieve evidence.

We would rather have the sort argued with than the individual items adopted. **The claim we
would defend is the meta one: most of what a first-time smart tool author gets wrong is
generatable, and the generator is where it should be fixed.**

---

**Provenance.** Tools:
[colombod/amplifier-smart-tools-research](https://github.com/colombod/amplifier-smart-tools-research).
Every defect above, with its measurement, is in
[`findings/colombod-building-two-smart-tools.md`](../findings/colombod-building-two-smart-tools.md).
