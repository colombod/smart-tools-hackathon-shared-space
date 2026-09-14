# Proposal: the intelligence contract should carry what the tool DID, not only what it SAID

**To:** [DavidKoleczek/amplifier-smart-tool-creator](https://github.com/DavidKoleczek/amplifier-smart-tool-creator)
**From:** colombod
**Evidence: MEASURED + OBSERVED** — the failure this prevents actually happened to us, twice,
in shipped code. The proposed schema itself is **not implemented**; read it as an argument.

## The contract today

```python
class Intelligence(Protocol):
    implementation: str
    def preflight(self) -> None: ...
    def run(self, request: AgentRequest) -> AgentResult: ...
```

This models *"I asked something and text came back."* It is the right shape for an agent SDK,
and we would not change the seam. The problem is what `AgentResult` can hold.

## The case that does not fit

We built two smart tools whose evidence comes from Perplexity. It is **not an agent runner**:
it owns its own search loop, takes no system prompt, exposes no tool list, and gives us no
turn to control. We send a question; it searches and fetches on its own; it returns:

```json
{ "text": "...",
  "sources": [ {"id": "s1", "url": "...", "title": "..."}, ... ],
  "usage": {
    "cost": { "total_cost": 0.0166, "currency": "USD",
              "tool_calls_cost_details": {"fetch_url": 0.01017, "search_web": 0.0025} },
    "tool_calls_details": {"fetch_url": {"invocation": 1}, "search_web": {"invocation": 1}} } }
```

Three things come back and **two of them have nowhere to go** in `AgentResult`:

- **`sources` as structured data.** Flattened into `output`, citation validation degrades into
  parsing prose for URLs. Our whole citation spine — every marker must resolve to a gathered
  source, or the document is rejected and repaired — needs these as a list with stable ids.
- **Per-call accounting.** What it searched, what it fetched, and what each cost.

## Why the second loss is not hypothetical

**We shipped exactly that bug.** We read a flat `cost_usd` field that does not exist on that
response shape, found nothing, and recorded `null` — so every Perplexity call was invisible in
our own accounting while the caller was genuinely being billed. A comment in our code
explained that `null` "means unreported", documenting our parsing error as a property of
somebody else's service. We then published that as a finding. It was wrong in both places.

A contract with a place to put per-call cost would have made the omission obvious.

## The proposal

```python
@dataclass
class Call:                                   # one thing the intelligence actually did
    name: str                                 # "search_web", "fetch_url", "read_file"
    invocations: int
    cost_usd: str | None                      # None means UNREPORTED, never free
    reported: Literal["live", "after"]        # per call as it happened, or once at the end

@dataclass
class AgentResult:
    output: ... | None
    error: ... | None
    evidence: list[Source] | None             # what the answer rests on
    activity: list[Call]                      # what it did to get there   <-- the new part
    usage: Usage
```

**One Protocol still covers both implementations.** An agent fills `activity` live, per tool
call. A service that owns its own loop fills it once, afterwards, with `reported="after"`.

That field settles something we spent real time getting wrong: the interesting variable was
never *whether* spend is reportable, but **when it becomes knowable**. An interface modelling
only a final total cannot express live spend; one modelling only a stream cannot express a
one-shot service. `reported` lets a single type be honest about both.

## Why `activity` is the load-bearing field

**An empty `activity` is how you detect a fabrication.**

The single most repeated defect in our project — four separate occurrences — was a capability
**declared** present and never actually exercised, with nothing noticing:

```
provider credential present      →  but no client library installed
tool in the mount plan           →  but never loaded
guard checking dependencies      →  checking a list that rots
guard reading the mount registry →  reading one that does not exist yet
```

A research agent whose search silently failed **does not fail**. It answers fluently from
memory and returns URLs it never opened. One of our live runs wrote *"No live access to the
two listed sources"* in its own brief while two sources sat in the record — fabricated
evidence in the voice of caution, passing every check we had, because a source was *present*.

The check that finally held asks the only question that cannot rot: **did the work actually
happen?** With `activity` in the contract, that stops being bespoke code in one tool and
becomes a property every scaffolded tool gets for free:

```python
if not result.activity:
    raise NoEvidence("nothing was searched or fetched")
```

**A scaffold shipping that as a default would inoculate every tool built from it against the
failure mode that cost us the most.** That is the real argument for putting this in the
creator rather than in a document: a scaffold settles conventions by default, and whatever
`init` emits becomes the norm long before any spec catches up.

## A smaller, separable change

**`preflight()` raises; it should also be able to report.** Ours returns structured
per-requirement state with provenance:

```json
{"name": "perplexity",  "state": "satisfied", "detail": "resolved from $PERPLEXITY_API_KEY"}
{"name": "ai-provider", "state": "satisfied", "detail": "$ANTHROPIC_API_KEY, client library present for anthropic"}
```

A host can read that **before deciding to call anything**, without catching an exception.
Raising is right for the library's own guard; a host needs the data form too. Note the second
line — credential present *and* client library present are different facts, and we learned
that distinction by shipping a tool that had the first and not the second.

## What we are not proposing

- **Not a second Protocol.** We previously argued acquisition and reasoning are two seams. On
  reflection, one seam with somewhere to put evidence and activity covers our case, and one
  type is worth more to the ecosystem than an accurate taxonomy.
- **Not changing `preflight()`'s signature.** Add a reporting form beside it.
- **Not the single-root layout.** Unrelated to this, and our two-roots-over-one-library shape
  is deliberate.

## Provenance

Both tools, the live runs, the accounting bug and its correction are in
[colombod/amplifier-smart-tools-research](https://github.com/colombod/amplifier-smart-tools-research).
The fuller write-up, including the measured A/B on `--help`-as-skill that led us to adopt the
creator's own design, is in
[`findings/colombod-building-two-smart-tools.md`](../findings/colombod-building-two-smart-tools.md).
