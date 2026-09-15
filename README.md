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
evidence/    artifacts a finding rests on — transcripts, outputs, run directories
```

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

---

## Proposals

Concrete changes, addressed to whoever owns the thing being changed.

| File | To | What it asks for |
|---|---|---|
| [`proposals/intelligence-contract-carry-what-it-did.md`](proposals/intelligence-contract-carry-what-it-did.md) | [smart-tool-creator](https://github.com/DavidKoleczek/amplifier-smart-tool-creator) | `AgentResult` should carry `evidence` and `activity` — what the intelligence DID, not only what it said. An empty `activity` is how a tool detects that its own search silently failed |
| [`proposals/output-is-bigger-than-the-response.md`](proposals/output-is-bigger-than-the-response.md) | the spec | A smart tool's output routinely exceeds its response and the caller's context. Proposes one `parts` vocabulary covering size, time and fidelity — a proxy you can act on, a cost-declared fidelity ladder, affordances with meaning, and a growing/final/**dead** distinction for detached calls |

---

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
