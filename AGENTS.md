# Agent instructions — Smart Tools hackathon shared space

You are working in the hackathon team's shared findings repository. Read this before
writing anything here.

This repository holds **what building taught us about the Smart Tools spec**. It is not
documentation for any tool, and it is not a place for plans or status.

---

## The one rule

**A finding carries evidence, or it is not a finding.**

Evidence is a command and its output, a file and line, a test that fails, a conformance
verdict, a transcript. "I think the spec should…" with nothing behind it is an opinion, and
opinions are what this repository exists to replace. The hackathon's own stated bias: a
spec opinion with no implementation behind it is not a deliverable.

If you are asked to record something you cannot back, **say so and record the gap** rather
than writing it as though you could.

### Open every section with its evidence tier

Not all evidence is the same strength, and a reader cannot tell by looking. We added these
after noticing that nine of eleven sections in the first findings file carried no measurement
and still read with exactly the same authority as the two that did — which is the failure our
own tools exist to prevent, committed in the document arguing for preventing it.

So each `##` section opens with one line:

```
**Evidence: MEASURED** — live before/after on <fixture>, figures from <path>. N=1 per arm.
```

| tier | means |
|---|---|
| **MEASURED** | a before/after or A/B you actually ran, with the figures **and their sample size** |
| **OBSERVED** | something you read in source or watched happen; true, but not a measurement |
| **JUDGMENT** | an argued call. Show the reasoning; say that the other arm was not built |
| **PROPOSAL** | a design not implemented or measured. An argument, nothing more |

**Always attach the sample size to a MEASURED claim.** Almost everything any of us can run in
a hackathon is N=1 per arm, and a figure quoted without that turns into a rate somebody else
will plan against.

A PROPOSAL section is welcome — it is how a design gets argued — but label it, so nobody
quotes it as a result.

---

## What belongs here

Things **about the spec**, learned by building against it:

- A rule the spec states that nothing checks.
- A responsibility the spec assigns and never returns to.
- A question the spec is silent on that a real tool had to answer anyway.
- A convention someone tried, with evidence of it working or not working.
- Friction on a paved path — what following the examples actually costs.

## What does NOT belong here

- **Documentation for a tool.** A tool's vision, architecture, contracts, README and
  manifest live in that tool's own repository. Never park them here, and never copy them
  here "for visibility".
- **Plans, task lists, status updates.** Work tracking lives in the work tracker.
- **Anything with a secret in it.** This repository is private, which is not the same as
  safe. No API keys, no tokens, no credential values — not even a prefix or a length.
  Transcripts and outputs go in `evidence/` only after they have been read for secrets.

---

## Layout and naming

```
findings/<who>-<what-you-built>.md    one file per person per body of work
proposals/<short-name>.md             a concrete change to the spec or conformance kit
evidence/<who>/<whatever>             artifacts a finding rests on
```

**One file per person per body of work.** Not a shared document. This is deliberate: nobody
has to merge prose with anybody, and a finding stays attached to whoever can answer
questions about it.

**Never edit another person's findings file.** If you disagree with one, or have evidence
that contradicts it, write your own file and reference theirs. Their name is on it; your
correction should have yours on it.

**Add a row to the index in `README.md`** when you add a file. A findings directory nobody
can navigate is a findings directory nobody reads.

---

## Writing a finding that survives its author

The reader is someone who was not there and **cannot ask you a follow-up**. Upstream has
issues, discussions and pull requests all disabled, so this reaches the spec's authors
through a person — there is no thread in which a terse observation gets repaired. Every
finding has to stand on its own.

Each file opens with:

```markdown
# Findings: <what this is about>

**Who:** <github handle>
**What this came from:** <the thing you built, with a link>
**Status:** <how current this is, and what is still only partial evidence>
```

Then, for each finding:

- **Observation before conclusion.** What you saw stays true even if what you concluded
  from it turns out wrong. Someone rereading this in a month needs the observation.
- **The evidence, inline.** The command, the output, the file:line. Short enough to read,
  specific enough to re-run.
- **Your own limits, named.** "This is install-time evidence; use-time evidence needs X" is
  more useful than a confident claim that quietly rests on less than it seems to. A finding
  that admits where it is thin tells the next person where to push.
- **Separate "the spec should change" from "we learned something".** Both are worth having
  and they go to different audiences. A proposal aimed at the conformance kit belongs in
  `proposals/`; a lesson about how to build belongs in your findings file.

### Do not

- Do not restate the spec back at itself. Assume the reader knows it.
- Do not soften a negative finding into a suggestion. "We failed this rule for three
  milestones and nothing caught it" is the useful sentence; "it might be worth checking
  this" is not.
- Do not present a single instance as a pattern. If you saw it once, say you saw it once.
- Do not fabricate a verdict, a count, or a test result. If the number is not in front of
  you, go get it or leave it out.

---

## Proposals

A proposal is a finding that has become specific enough to act on. Put it in `proposals/`
and give it:

- the spec sentence or conformance rule it concerns,
- what would change,
- what it would cost to implement and check,
- and what evidence says it is needed — normally a link to a finding.

Order proposals by how cheap they are to adopt. A proposal nobody can estimate is a
proposal nobody adopts.

---

## Git

Small commits, one subject each. Write the commit message for a teammate skimming the log
to see whether anything new landed in their area — name the finding, not the file
operation. Do not rewrite shared history.

Commit footer, as elsewhere in this ecosystem:

```
Generated with Amplifier

Co-Authored-By: Amplifier <240397093+microsoft-amplifier@users.noreply.github.com>
```
