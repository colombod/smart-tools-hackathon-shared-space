# Proposal: a smart tool's output is usually bigger than its response

**To:** the Smart Tools spec · **From:** colombod
**Addresses ROADMAP #1** (long-running / expensive calls) and the unasked question next to it:
results too large for the caller that requested them.

**Evidence: MIXED.** The problem statement is **OBSERVED** — our own tools hit it on ordinary
runs, with figures below. The `parts` vocabulary is **PROPOSAL**: not implemented, not
measured. Where we already ship a partial version, it is named as such.

---

## The problem the spec does not name

A smart tool wraps expertise. The useful ones produce **more output than a caller can hold**:

| tool | what it produces |
|---|---|
| research | a 78-line report over 54 sources, 5m10s, $0.31 — measured, `dr-56f6e5ec` |
| transcription | an hour of audio → tens of thousands of words |
| image generation | a 20MB PNG |
| video render | a file measured in gigabytes |
| dataset generation | rows no context window will ever hold |

The spec is silent on this, so **every tool author invents an answer**, and the default
invention is *return everything and hope*. That fails twice over: it floods the caller's
context, and it forces the tool to finish before saying anything at all.

Both failures have the same cause. **The response is treated as the output.** It is not. The
response is what the caller can *act on*; the output is what the tool *produced*, and the two
have never been the same size.

## One vocabulary, three axes

A response is **a sequence of parts**. Part one is a *proxy*: immediately useful, bounded by
design, and enough to decide what to do next. Later parts refine, extend, or complete it.

That single idea covers three problems we were about to solve separately:

```
SIZE      the whole is too big for context      →  ask for the parts you need
TIME      the whole is not finished yet         →  take the parts that exist, rejoin for the rest
FIDELITY  a part exists at several resolutions  →  thumbnail and original are the SAME part
```

Designed apart, these become a navigation block, a polling URL and a quality parameter that
know nothing about each other. Designed together, a caller learns one vocabulary and a tool
author implements one thing.

### This is HLS, and that is the point

The video case is already solved, in public, for twenty years. **An HLS or DASH manifest is
literally a sequence of parts at multiple fidelities**, refreshed while a live stream is still
being produced and marked final when it ends.

The property worth stealing: **a live manifest and a finished one are the same document type.**
The only difference is a flag saying whether more parts are coming. A consumer writes one
reader, not a streaming path and a batch path. We would rather inherit that than reinvent it
worse.

## What a tool returns

Four things. The names are illustrative; the obligations are the proposal.

**1. A proxy — immediately useful, bounded, honest about being a proxy.**

Not a teaser and not a truncation. A brief that answers the question; a thumbnail you can
actually look at; a transcript slice you can read. If the caller needs nothing more, it is
done.

**2. A fidelity ladder — the rungs that exist, and what each costs before you climb.**

```
thumbnail  →  web resolution  →  original
brief      →  section         →  full report  →  raw sources
transcript →  clip            →  master audio
```

Cost in **bytes and money**, declared up front. A caller that cannot find out what a fetch
costs before making it has no way to protect itself.

**3. Affordances with meaning — named actions, not URLs and not bare strings.**

Each says what it does, what it returns, what shape comes back, what it costs, and whether it
needs credentials. **We ship a weak version today**: a `next` block of shell command strings —
useful to an agent driving a CLI, meaningless to one calling the library, and silent about
size and cost. It works, and it is not enough.

**4. Completeness, always — as a property of the response type.**

Every part says whether it is whole or partial and how much sits beyond it. Ours does this for
one verb:

```json
{"complete": false, "returned_lines": 60, "total_lines": 78,
 "note": "read is INCOMPLETE: 18 lines sit beyond this window..."}
```

That is the right behaviour in the wrong place — it should be a property of every response,
not one verb's courtesy.

## Streaming is the same thing, along time

`--detach` returns **part one**: the identifier, where parts will appear, and an explicit
statement of what is not yet true. The caller leaves. It rejoins whenever it likes and reads
the manifest.

Three states, and the third is the one that matters:

```
GROWING              more parts are coming
FINAL                the sequence is complete
FAILED-OR-ABANDONED  nothing is coming, and here is what exists
```

**A status that reads `running` forever because the process died is worse than blocking.** It
is the exact silent-failure shape this project hit four separate times — something declared
in progress, with nothing checking whether it is true. If a spec adopts nothing else here, it
should adopt the requirement that a detached call can distinguish *working* from *dead*.

## Worked example: an image tool we have not built

The test of a general pattern is whether it survives content its author does not produce.

```jsonc
{ "result": {
    "id": "img-8f2e",
    "part": { "kind": "thumbnail", "media_type": "image/webp",
              "bytes": 18_400, "width": 320,
              "inline": "data:image/webp;base64,..." },   // small enough to carry
    "complete": false,
    "ladder": [
      {"rung": "thumbnail", "bytes": 18_400,    "cost_usd": "0.00", "ready": true},
      {"rung": "web",       "bytes": 240_000,   "cost_usd": "0.00", "ready": true},
      {"rung": "original",  "bytes": 21_400_000,"cost_usd": "0.00", "ready": true}
    ],
    "affordances": [
      {"name": "fetch", "does": "retrieve this image at a chosen rung",
       "params": {"rung": ["thumbnail","web","original"]},
       "returns": "bytes", "needs_credentials": false},
      {"name": "describe", "does": "caption the image",
       "returns": "text", "cost_usd": "0.004", "needs_credentials": true}
    ] } }
```

Nothing strains. The proxy is a thumbnail instead of a brief; the ladder is resolutions instead
of report sections; one affordance is free and one spends. **An agent can decide whether it
needs 21MB without fetching 21MB** — which is the whole point, and is impossible today.

For video, part one is a keyframe and the ladder is segments; the response is a manifest that
grows while the render runs. That is HLS, and it needs no new invention.

## The failure we are fixing in our own tool first

From surveying hypermedia (`dr-56f6e5ec`), the sharpest transferable lesson was about
`204 No Content`: **a response with no representation carries no way onward.**

**Our refusals have exactly that defect right now.** A run that fails with `NoEvidence` returns
an error envelope and strands the caller — no partial evidence, no suggestion, nothing. A
refusal is a response, and a response owes the caller a next move.

## What this asks of the spec

1. **Name the problem.** A tool whose output can exceed a caller's context should say so and
   say what it does about it. Today nothing prompts an author to think about it at all.
2. **Adopt a part/proxy vocabulary** rather than leaving each tool to invent one, so a host
   learns it once.
3. **Require cost and size before commitment** on anything a caller can fetch.
4. **Require a growing/final/dead distinction** on anything detachable. This one is not
   ergonomics — without it, async is strictly worse than blocking.
   *To be precise about what is being asked:* the obligation is that a caller can find out
   whether work is still happening. **How a tool spells that is its own business** — we are not
   proposing `--detach` as a standard flag, and we have one implementation, which is nowhere
   near enough to standardise a name from. The line we would draw: standardise what **arrives**
   (response fields a host must parse), not what you **send** (flags, which every smart tool
   already documents for itself in its skill).
5. **Require a refusal to carry affordances**, for the same reason `204` was a mistake.

And a conformance check that would have caught our own defect: **every response a tool can
return, including errors, must carry at least one way onward.**

## What we have not earned yet

The `parts` vocabulary is **not implemented** — this document is an argument. The image example
is a sketch, not a shipped tool. Our completeness block, `next` block, durable run directory
and free deterministic navigation verbs are real and in use, and they are a *partial,
text-only* version of what is proposed here.

We would rather have this argued against before we build it than build it and discover the
vocabulary only fits research.

---

**Provenance.** Tools:
[colombod/amplifier-smart-tools-research](https://github.com/colombod/amplifier-smart-tools-research).
The hypermedia survey behind the design choices, including what we explicitly rejected, is in
[`findings/colombod-building-two-smart-tools.md`](../findings/colombod-building-two-smart-tools.md).
