# A smart tool that measures its own staleness: what a knowledge tool does to the spec

**Who:** colombod
**What this came from:** building
[`lore` 0.1.0](https://github.com/colombod/amplifier-smart-tools-lore), a smart tool that
answers "how do I use X" and "how does X work" from DeepWiki and Context7, to replace
[`amplifier-bundle-deepwiki`](https://github.com/colombod/amplifier-bundle-deepwiki), a
bundle doing the same job through MCP. 16/16 conformant, 186 tests, exercised in a clean
container with no credentials and driven from Claude Code, Codex and the Amplifier CLI.
**Status:** the tool is built, published and measured. The three spec arguments below rest on
N=1 tool each, and I say so where it matters. The staleness figures move between runs by
design, so every one is stamped with when it was taken.

A knowledge tool is a bad fit for the spec in two specific places, and both only showed up
because the domain has a property the other tools I have built do not: **its answers have an
age, and its payloads do not fit.** Everything below is what those two properties did to a
conformant tool.

---

## A single conformant tool response can be 292,000 tokens, and nothing notices

**Evidence: MEASURED** - one `read_wiki_contents` call against the public DeepWiki MCP
endpoint, captured to disk and counted. N=1 repository, re-measured across the session with
identical figures.

```
repository                 upstash/context7
raw JSON-RPC response      2,412,495 bytes
markdown carried in it     1,169,101 characters
pages inside that markdown 130
estimated tokens           ~292,000
wall clock                 1.7 s
```

That is one call, for a mid-sized repository, returning roughly 292,000 tokens. Nothing in
the spec bounds a capability's response size, and the conformance kit has no rule that could
see it: `lore` passes 16/16 whether it hands that back or writes it to disk.

The bundle this tool replaces states in its own guidance that the content is "automatically
truncated to ~50k chars". **That is not true at the MCP layer.** 1,169,101 characters came
back. The belief was plausible, written down, and wrong, and nothing between the belief and
the caller checked it.

So the design the domain forced: `fetch` writes the wiki to disk as one file per page plus an
index, and returns the index. `read` and `search` are the only ways back in, and both are
capped and report what they withheld. Measured in the container:

```
lore fetch upstash/context7     1,165,517 characters to disk, 3,315 bytes to the terminal
lore read <repo> 1 --limit 500  501 bytes out, against 7,708 unbounded
```

The bounded read was checked as a strict byte prefix of the unbounded one, not by trusting the
`truncated` flag. A 500-byte answer is equally consistent with a tool that truncates and one
that re-renders a shorter summary; only the prefix comparison excludes the second.

**For the spec:** nothing says a capability should bound its own output, and for most domains
nothing needs to. For a tool whose payload is somebody else's generated corpus, the caller's
context window is a real constraint that the tool, not the caller, has to respect. This is
adjacent to ROADMAP question 1 (long-running and expensive calls) but is not the same
question: this call was fast. It was *large*. `proposals/output-is-bigger-than-the-response.md`
already argues this case and proposes a `parts` vocabulary for it. The figures above are one
more live data point under that proposal rather than a competing one.

## The freshness signal is sitting there in the page, and asking a model for it instead is a choice

**Evidence: MEASURED** - the indexed commit scraped from `deepwiki.com`, compared against the
GitHub compare API. N=1 repository, sampled four times across roughly two hours.

DeepWiki's page renders, in the HTML:

```
Last indexed: <!-- -->20 July 2026<!-- --> (<a href=".../commits/23843e9c">23843e</a>)
```

A date and a commit SHA. Note the anchor text is truncated to six characters while the href
carries more, so the link text is the wrong place to read it from. Against
`GET /repos/upstash/context7/compare/23843e9c...master`:

| when, 2026-09-22 | head | commits behind | days behind |
|---|---|---|---|
| ~14:30 UTC | `eb27b949` | 82 | 63 |
| ~16:10 UTC | `06b320a3` | 83 | 64 |
| ~16:30 UTC | `ea6b3d8c` | 84 | 64 |

The gap moved three times in two hours because it is a live measurement rather than a stored
opinion. The bundle this replaces obtained the same signal by **asking DeepWiki's model**
"what is the latest version of this library covered in your documentation?" and grading the
reply into LOW/MODERATE/HIGH/UNKNOWN. That is a model's self-report about its own index,
which is the one witness with no independent access to the fact.

Not every repository has the signal: `tiangolo/fastapi`, `microsoft/amplifier-smart-tools` and
one of my own repositories all render a ~31KB shell page with no marker, and the MCP endpoint
agrees, answering `isError: true` with "Repository not found. Visit
https://deepwiki.com/<owner>/<repo> to index it." Two independent signals agreeing that a
repository is unindexed is a different and far more actionable state than a parse failure,
and it is worth reporting as itself.

**For the spec:** this is not a rule the spec is missing so much as a shape it has no vocabulary
for. Several of the tools in this hackathon wrap a third-party index, and **an answer from an
index has an age that the tool can often measure and the caller cannot see.** There is no
manifest field, no conformance rule, and no convention that asks a tool whether its answer is
current. I am not proposing a required field on N=1. I am saying the question has never been
put, and every knowledge tool we ship will have to answer it privately.

## Both defects that mattered were invisible to the tests and to the reviewer

**Evidence: MEASURED** - two defects, found by running the tool rather than testing it, after
164 passing tests, 16/16 conformance and a clean model-backed review of the same code. N=2
defects.

**One.** `store.read_page` handed back a continuation telling the caller how to reach the rest
of a page:

```
lore read upstash/context7 --page 1 --start 2000
```

The CLI takes the page as a positional argument. `--page` is not an option, so the one string
whose entire job is to be runnable was rejected by the CLI. A unit test asserted the string's
*content* and passed. The fix was to assert its *executability*: `tests/test_continuation_runs.py`
now `shlex.split`s the continuation and invokes it against the real Typer application.

**Two.** `howto` given caller-supplied material still resolved the library name against
Context7 first, then attached that repository's staleness to the answer. Given an internal
name, Context7 returned an unrelated public library and the answer carried:

```
CAVEAT: The index's age for baybreezy/ui-thing could not be measured ...
```

for an answer drawn entirely from the caller's own text. Every claim in the answer was correct.
The provenance named the wrong repository. Caller-supplied material now resolves and retrieves
nothing.

Both defects live at the seam between two components that were individually correct and
individually tested. This is the third time across three tools I have watched that specific
pattern produce the only defects that mattered, and it is the strongest argument I have for
why *rendering the output and measuring it* belongs in the audit ladder above conformance and
above model-backed review.

## `check-spec-adherence` is a sampling process, and its headline line can mislead

**Evidence: MEASURED** - three full runs against `lore` across the session, each after the
previous run's findings were fixed. N=3 runs, one tool. Logs at
`evidence/colombod/spec-adherence/lore-spec-adherence{,-2,-3}.log`.

```
run 1   11 adhere,  6 deviate
run 2   15 adhere,  2 deviate     after fixing all 6
run 3   13 adhere,  4 deviate     after fixing both
```

Deviations went up after being fixed, and none of run 3's four were among run 1's six. The
classes recur; the instances differ, because each run samples different files. This now holds
across five tools (`aud`, `vid`, `deep-research`, `fact-check`, `lore`) and ten runs, and
**not one run has ever come back clean.** Nobody has yet established whether it terminates.

A second, more costly property: **the per-check summary line prints only the FIRST evidence
item.** Three of run 3's four deviations led with a positive observation:

```
deviates capability-skills-complete: src/lore/core/skill.py:13-73 registers all ten
  exposed capabilities, and each points to its rendered skill source
```

which reads as a pass. The actual defect was in evidence items 2 through 4. A reader skimming
headlines would have closed three real findings as noise. Read all the evidence, never the
headline.

For what it is worth, the findings themselves were good. Run 1 found a genuine crash I had not:
`gh api` output passed straight into `json.loads` while the CLI caught only its own error type,
so malformed output escaped as an uncaught traceback.

## The spec says the payload is data, not a reference. A knowledge tool cannot fully comply

**Evidence: JUDGMENT** - an argued call from one tool. The other arm, a version that takes
content only, was not built.

The spec is clear:

> **The payload is data, not a reference.** At the library level, a caller passes the actual
> content. This keeps the library free of assumptions about where the caller's material lives
> and keeps it usable from processes that have no filesystem in common with the caller.

The reason is sound and I do not want it weakened. But a knowledge tool's entire value is that
it *goes and gets* material the caller does not have, and the material is the 292,000 tokens
above. Requiring the caller to pass it as data means requiring the caller to hold it, which is
the exact problem the tool exists to solve.

What I did: `explain` and `howto` take an optional `material` parameter, so a caller that has
content passes content and nothing is retrieved, while the reference form stays the default.
The CLI's `--material-file` reads a path and passes the contents, keeping the path convenience
on the CLI where the spec puts it. That satisfies the rule without pretending retrieval is the
caller's job.

The unresolved part, and I state it as unresolved: with caller-supplied material the tool
cannot measure the age of anything, so its freshness verdict is `unknown` naming exactly why.
**Data-in gets you portability and costs you provenance.** I do not think the spec has noticed
that trade yet, and I have one tool's worth of evidence for it, not a pattern.

## Three hosts, one tool, and the age survived all three

**Evidence: MEASURED** - the same prompt through Claude Code 2.1.263, Codex CLI 0.154.0 and
the Amplifier CLI, with the agent skill installed by `npx skills add` and the tool installed
by `uv tool install` from the public repository. N=1 prompt per host.

The prompt asked for the Context7 search endpoint and whether a key is required, and said not
to answer from memory. All three ran the tool, and all three carried the measured staleness
into their own answer unprompted: Claude Code rendered a two-row source-age table, Amplifier
rendered the indexed and live SHAs side by side, and Codex consumed `--json` and quoted the
`commits_behind` and `verdict` fields. Nothing about the hosts was configured beyond installing
the skill.

This is the cheapest evidence I have for ROADMAP question 3. One `smart-tool.json`, one agent
skill, and `uv tool install` was the whole integration for three different hosts, none of which
know anything about each other. **Worth noting what did the work: it was `--json`, not the
skill.** Codex's answer is the most faithful of the three because it read structured fields
rather than parsing prose.

One host-level observation, not a tool defect: Claude Code reconciled its two sources rather
than picking one, reporting that Context7's documentation says a key is required while the
tool's own live check reached the endpoint without one. Both are true. A grounded tool buys
**source fidelity, not ground truth**, and that distinction deserves to be said out loud more
often than it is.

## What I could not settle

- **Whether `check-spec-adherence` terminates.** Ten runs, five tools, never clean. Three runs
  on a stabilising tool went 6, 2, 4. Someone should run it five or six times on a tool nobody
  is touching and find out.
- **Whether measured freshness changes behaviour or just decorates it.** Every answer now
  carries a verdict, and all three hosts repeated it. Nobody has measured whether a caller
  holding a `stale` verdict actually verifies anything it would otherwise have trusted. That
  is the experiment that would justify the whole design, and I did not run it.
- **Whether the bundle was the wrong shape or just an early one.** I replaced an MCP bundle
  with a smart tool and the tool is better on context and on freshness. I did not build the
  bundle version of the disk-and-cap design, so I cannot claim the bundle form could not have
  got there.
- **The Copilot entitlement gap.** The spec says a missing prerequisite fails immediately.
  Missing `gh` does. A missing Copilot subscription cannot: the SDK's own end-to-end suite
  skips `account.getQuota` as "defined in schema but not yet implemented in CLI", and
  `models.list` and `auth.getStatus` confirm GitHub auth rather than entitlement. So the tool
  retrieves first and fails at the model call, and I documented that rather than inventing a
  check. If other tools hit the same wall, the spec's "fails immediately" may need to
  distinguish prerequisites a tool can check from ones it cannot.
