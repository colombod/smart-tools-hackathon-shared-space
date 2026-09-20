# Proposal: the conformance kit should check the agent skill it tells you to ship

**To:** [microsoft/amplifier-smart-tools](https://github.com/microsoft/amplifier-smart-tools)
(conformance kit)
**From:** colombod
**Evidence: MEASURED** — the failure this prevents was live in 3 of our 4 shipped skills,
found by a user's harness rather than by any of our CI. N=4 skills, 1 host log. The proposed
rule is **implemented twice** — as a regression test in `amplifier-smart-tools-audio` and in
`amplifier-smart-tools-research` — but **not** in the kit; that part is the ask.

## What happened

A harness printed this while discovering skills on a user's machine:

```
Skill 'fact-check'    ... exceeds 1024 character description limit (1148 chars). Continuing with discovery.
Skill 'aud'           ... exceeds 1024 character description limit (1093 chars). Continuing with discovery.
Skill 'deep-research' ... exceeds 1024 character description limit (1254 chars). Continuing with discovery.
```

Measured across every skill our four tools ship:

| skill | frontmatter `description` | limit |
|---|---|---|
| `deep-research` | 1254 | 1024 |
| `fact-check` | 1148 | 1024 |
| `aud` | 1093 | 1024 |
| `vid` | 829 | ok |

**Three of four.** All three had passed `check-conformance` 16/16 at the time, repeatedly,
across many releases.

## Why the kit missed it, and why that is the kit's problem anyway

The 1024-character cap belongs to the **Agent Skills** specification, not to the Smart Tools
spec. So on a narrow reading this is out of scope.

But the Smart Tools spec is what tells you to ship a `SKILL.md`, and it is the reason that
file exists in the repository at all. `agent-skill-is-thin` is already a spec check — the kit
and the reviewer both have opinions about that file's *contents*. Declining to check the one
hard numeric limit the file must satisfy leaves the tool author to discover it from a user.

Two properties make it worse than an ordinary bug:

- **It degrades silently.** *"Continuing with discovery"* — the host does not refuse the
  skill, it proceeds with truncated or dropped metadata. A tool is then partially
  discoverable and nobody is told which part was lost.
- **It surfaces where no author is watching.** In the host, on someone else's machine, at
  discovery time. Ours was caught only because a human pasted a log into a chat. Nothing in
  any of the three repos' CI would ever have gone red.

## The rule

One deterministic check, no model, no network, no invocation of the tool:

```
skill-description-within-limit
  For every skills/*/SKILL.md in the distribution root:
    parse the YAML frontmatter
    FAIL if len(frontmatter["description"]) > 1024
  SKIP when the tool ships no skills/ directory.
```

Two implementation notes that cost us time:

- **Measure the parsed value, not the raw bytes.** These descriptions are written as YAML
  block scalars (`>-`), so folding changes the length. Counting characters in the file gives
  a different — and wrong — answer from counting the loaded string.
- **Glob the directory; do not name the skills.** A tool that adds a second skill later is
  the case the rule is protecting, and a hardcoded list would not cover it.

Our version, in both repos, parametrises over `skills/*/SKILL.md` and asserts the limit. It
is about ten lines and runs in milliseconds.

## Why the kit rather than a template

A scaffold could emit a short description and a lint rule, but descriptions are edited after
scaffolding — every one of our three drifted over the limit during normal authoring, as
capability lists and trigger phrasings accreted. The limit binds at *publish* time, which is
where the kit already stands.

## What it does not solve

The cap is a **length** rule, and length is the least interesting property of a description.
The description is the discovery surface: the only thing an agent sees when deciding whether
to reach for the tool, before any help has been read. Ours were over the limit *because* they
had drifted into enumerating capabilities that runtime help already owned. Cutting them back
to 713 / 784 / 804 meant deleting capability lists and measurement inventories and keeping
the distinctive trigger phrasings and the non-goal clause — because a boundary ("this is
mastering, not mixing; not video, not stems, not transcription") does more selection work
than another verb name.

No mechanical rule can check *that*. A separate finding of ours records the related bug where
all three tools' descriptions matched the tool's **name** rather than the user's **intent**,
which no length check would have caught either. This proposal is only for the hard limit —
the part that is already numeric, already violated, and already reported by hosts in a
message no tool author ever sees.
