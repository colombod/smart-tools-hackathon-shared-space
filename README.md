# Smart Tools hackathon — shared space

Where we pool what we learn while building against the
[Smart Tools spec](https://github.com/microsoft/amplifier-smart-tools).

The hackathon has two goals: expand the ecosystem with real tools, and resolve the spec's
open questions. This repository serves the second one. It is not documentation for any
tool — each tool documents itself in its own repository.

**Why a repository rather than issues upstream:** `microsoft/amplifier-smart-tools` has
issues, discussions and pull requests all disabled, and its CONTRIBUTING says it is not
currently accepting external contributions. There is no inbox to file against, so findings
reach the spec's authors through a person. This is where they wait in the meantime, in a
shape someone can read without having been there.

---

> **Working here with an AI agent?** [`AGENTS.md`](AGENTS.md) carries the conventions in a
> form your agent will pick up automatically: the evidence bar, the naming rule, how to
> write a finding that survives its author, and what not to do.

## What belongs here

Things that are **about the spec**, learned by building:

- A rule the spec states and nothing checks.
- A responsibility the spec assigns and never returns to.
- A question the spec is silent on that a real tool had to answer anyway.
- A convention we tried, with evidence of it working or not.
- Friction on a paved path — what following the examples actually costs.

Things that do **not** belong here:

- Documentation for a tool you built. That goes in the tool's own repository.
- Opinions with no implementation behind them. The hackathon's whole bias is that a spec
  opinion with nothing built behind it is not a deliverable.

---

## What we built

Every finding here comes out of one of these. All three are in the
[catalog](https://github.com/microsoft/amplifier-smart-tools-catalog).

| tool | what it does | repo |
|---|---|---|
| **`vid`** | Video editing and curation — trim, stitch with transitions, retime, caption, grade, narrate. Finds a moment by what was SAID or SHOWN. 21 verbs, chainable into one ffmpeg pass. | [amplifier-smart-tools-video](https://github.com/colombod/amplifier-smart-tools-video) |
| **`deep-research`** | Researches a question across many sources; returns a brief plus the evidence on disk. | [amplifier-smart-tools-research](https://github.com/colombod/amplifier-smart-tools-research) |
| **`fact-check`** | Checks claims independently and returns a verdict per claim with its sources. | [amplifier-smart-tools-research](https://github.com/colombod/amplifier-smart-tools-research) |

```bash
uv tool install 'vid[all] @ git+https://github.com/colombod/amplifier-smart-tools-video'
uv tool install 'git+https://github.com/colombod/amplifier-smart-tools-research#subdirectory=tools/deep-research'
uv tool install 'git+https://github.com/colombod/amplifier-smart-tools-research#subdirectory=tools/fact-check'
```

Both repos run CI on every push: tests, the external conformance kit, and an
install-from-git job that takes the path a real user takes. A finding that cites one of
these is citing something you can install and check yourself.

---

## Build the tool before you write about it

**[DavidKoleczek/amplifier-smart-tool-creator](https://github.com/DavidKoleczek/amplifier-smart-tool-creator)**
— a smart tool for building smart tools. Start here rather than hand-rolling a skeleton:

```bash
uv tool install git+https://github.com/DavidKoleczek/amplifier-smart-tool-creator
smart-tool-creator init my-tool --description "..." --skill
```

Measured on this hackathon: **5.2 seconds**, 27 files, committed git repository, environment
synced, an Agent Skill written, the spec and SDKs shallow-cloned into `reference/` — and
**15/15 on the conformance kit with zero domain code written**. Doing the equivalent by hand
took a full milestone, and the write-up in
[`findings/colombod-building-two-smart-tools.md`](findings/colombod-building-two-smart-tools.md)
records both what it collapses and where it did not fit a two-root tool.

Getting to conforming is the cheap part now. **Findings come from what you build on top of
it**, so spend the time there.

---

## Layout

```
AGENTS.md    conventions, in the form an AI agent picks up automatically
findings/    one file per person per body of work
proposals/   concrete changes to the spec or the conformance kit
open-questions.md  what we did NOT settle, and what would settle it
presentations/  decks built from the evidence in this repo
evidence/    artifacts a finding rests on — transcripts, outputs, run directories
```

**Four kinds of thing live here, and they are not interchangeable.** A finding is what
building taught us, with the evidence attached. A proposal is a concrete change to the spec
or the kit. An open question is something we could not settle and a description of what
would settle it. And a negative result — a thing we tried that did not work — belongs in a
finding beside the positives, not quietly dropped: three of ours were worth more than the
features they came from.

Two standing rules worth knowing before you add anything: **never edit someone else's
findings file** — write your own and reference theirs — and **nothing with a secret in it**,
since private is not the same as safe.

**Name your file `findings/<you>-<what-you-built>.md`.** One file per person per body of
work, rather than a shared document, so nobody has to merge prose with anybody and a
finding stays attached to whoever can answer questions about it.

Add a row to the index below when you add a file.

---

## Index

| File | Who | What it came from |
|---|---|---|
| [`findings/colombod-building-two-smart-tools.md`](findings/colombod-building-two-smart-tools.md) | colombod | Converting an Amplifier bundle into two conforming smart tools (`deep-research`, `fact-check`) — [amplifier-smart-tools-research](https://github.com/colombod/amplifier-smart-tools-research) |
| [`colombod-building-a-video-smart-tool.md`](findings/colombod-building-a-video-smart-tool.md) | colombod | A video editing/curation tool: `requires[]` cannot express alternatives, closed sets as the safety line, generated code graded by arithmetic, and a no-provider test that passed for the wrong reason. Second round adds what a credential-free container found -- a shipped dead-pointer refusal, an over-stated credential requirement, and install identity vs install success. Third round, from a 21-verb surface: what makes a model-backed capability shippable, an intelligence seam more capable than its schema, four kinds of AI in one tool, and four defects only measurement found. Plus the discovery bug all three of our tools shipped: skill descriptions that matched the tool's NAME instead of the user's intent, found by a real user in Codex. |

---

## What the tools actually do, measured

Numbers rather than adjectives, so a reader can tell a claim from a hope. Every one is
reproducible from the repos linked above.

| claim | how it was checked | result |
|---|---|---|
| a generated ffmpeg transition works | rendered, then frames sampled mid-blend | 5 of 6 correct; the 6th **refused itself** |
| colour transfer matches a reference | Lab distance, before and after | **98.4%** of the gap closed |
| a narration line fits its slot | spoken audio measured against the slot's budget | 4.25s into a 6.00s slot |
| a vignette darkens the edges | corner and centre sampled separately | corner **37.3%** darker |
| retime lands on the duration it promises | container duration vs arithmetic | **exact** at 0.5x, 0.75x, 1.5x, 2x |
| deterministic paths need no credential | run in a container with none, asserted absent | holds — and **failed once** for the wrong reason |
| it installs the way a user installs it | `uv tool install` from git, in CI, every push | green in both repos |

The pattern under all of them: **the model writes, arithmetic judges.** Where a cheap
deterministic check exists on a model's output, generation is defensible. Where it does not,
it is a coin flip with good manners.

## Proposals

Concrete changes, addressed to whoever owns the thing being changed.

| File | To | What it asks for |
|---|---|---|
| [`proposals/intelligence-contract-carry-what-it-did.md`](proposals/intelligence-contract-carry-what-it-did.md) | [smart-tool-creator](https://github.com/DavidKoleczek/amplifier-smart-tool-creator) | `AgentResult` should carry `evidence` and `activity` — what the intelligence DID, not only what it said. An empty `activity` is how a tool detects that its own search silently failed |
| [`proposals/what-the-generator-should-ship-by-default.md`](proposals/what-the-generator-should-ship-by-default.md) | [smart-tool-creator](https://github.com/DavidKoleczek/amplifier-smart-tool-creator) | Everything that bit us, sorted by whether a generator can make it structural. Eight of ten are automatable — most of what a first-time author gets wrong is a missing default, not missing wisdom |
| [`proposals/long-running-is-the-hosts-problem-too.md`](proposals/long-running-is-the-hosts-problem-too.md) | the spec | ROADMAP q1, answered from three harnesses and two real detached runs. Two hosts hit the same wall and invented two different correct ways over it — so the spec must state the obligation and refuse to prescribe the mechanism. Plus: polling must be documented as free, liveness needs a terminal dead state, and cost must be declared with its uncertainty |
| [`proposals/output-is-bigger-than-the-response.md`](proposals/output-is-bigger-than-the-response.md) | the spec | A smart tool's output routinely exceeds its response and the caller's context. Proposes one `parts` vocabulary covering size, time and fidelity — a proxy you can act on, a cost-declared fidelity ladder, affordances with meaning, and a growing/final/**dead** distinction for detached calls |

---

## Still open

**[`open-questions.md`](open-questions.md)** — what we did not settle, ordered by how cheaply
a real answer could be had. Includes the honest gap in our loudest finding: we fixed every
skill description in all three tools and **never measured whether the fix works**.

Add to it rather than starting a new file. An open question with a named experiment beside
it is worth more than a finding nobody acts on.

## Writing a finding that survives you

The reader is someone who was not there and cannot ask you a follow-up.

- **Say what you observed before what you concluded.** The observation stays true if the
  conclusion turns out wrong.
- **Show the evidence.** The command, the output, the file and line. A finding with no
  evidence is an opinion wearing a finding's clothes.
- **Name what you are unsure of.** A finding that admits its own limits is more useful than
  one that overreaches, because the next person knows where to push.
- **Separate "the spec should change" from "we learned something".** Both are worth having
  and they go to different audiences.
