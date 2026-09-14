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

## Layout

```
findings/    one file per person per body of work
proposals/   concrete changes to the spec or the conformance kit
evidence/    artifacts a finding rests on — transcripts, outputs, run directories
```

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
