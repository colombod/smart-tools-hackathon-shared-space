# Converting an Amplifier bundle to a smart tool

You have the smart-tool spec; it tells you what to build. This tells you what
to carry across from the bundle, what cannot come, and how to know when you
are done.

**The bundle is the specification for behaviour.** The spec governs shape. If
the bundle does something, your tool does it, unless a fact forces otherwise
and you write that fact down.

---

## What maps to what

| In the bundle | In the smart tool |
|---|---|
| `modules/tool-*/` — the Python module and its client | The library, plus a thin CLI over it |
| Its tool descriptions and JSON schemas | Catalogue entries, **copied verbatim** — see [the-catalogue.md](the-catalogue.md) |
| `modes/<name>.md` prose — standing orders, when-not-to-use | `SMART_TOOL.md` body |
| `contributes.agents[].instruction` | System prompt of your model-backed capability |
| `context/*.md` — data model, endpoint reference, query patterns | `SMART_TOOL.md` body, as a reference section |
| `examples/settings.snippet.yaml` | `.env.example` and `CONFIGURATION.md` |
| Agent `description` — "MUST be used for" / "Do NOT use for" | `SMART_TOOL.md` body, "when not to use it" |

## What has no counterpart

These address a running Amplifier session. A library is called and returns; a
CLI process exits. There is nothing for them to act on.

| Bundle construct | Why it cannot come |
|---|---|
| The mode itself (`/name`, `shortcut`) | No session state to overlay |
| `tools.safe`, `default_action` | Amplifier gates tools; a library cannot gate its own caller |
| `contributes.agents` | Delegation forks a child session. Your model loop is a function call, not a sub-agent |
| `contributes.context` | Conditional loading — "only while the mode is active". You have no conditionality: text is in the manifest or it is not |
| `model_role` | A hint to Amplifier's model router |
| `includes:` in `bundle.md` | Composition of behavioural packages |
| `behaviors/*.yaml` | Module wiring and config injection from Amplifier's `settings.yaml` |

**Record these once, then stop worrying about them.** They are not gaps.

**But port what they carried.** Each of those mechanisms usually delivered
something. The mode delivered prose. The agent delivered an instruction. The
`contributes.context` entry delivered a reference document. Concluding "the
mechanism cannot port, so this is resolved" is how content gets lost — see
"Confusing the mechanism with what it delivered" in [traps.md](traps.md),
which cost three attempts to get right.

---

## Before you start

**Confirm which copy of the bundle you are reading.** There is usually more
than one — the repo you cloned, and `~/.amplifier/cache/<bundle>-<hash>/`.
They can differ. Statements about "what the bundle does" made from the wrong
one are worthless and hard to detect.

**Confirm which spec repo is authoritative.** More than one exists, and one is
a predecessor. Check which has the complete file set, and fetch before
reading — a stale checkout produces conclusions that were true last month.

**Inventory the bundle, file by file, with line counts.** This is what later
tells you whether you are done. Without it, "did we carry everything?" is
answered from memory, and memory misses the largest, least code-like file. A
404-line context document was missed entirely on one first pass — bigger than
any source file in the bundle, and the most valuable thing in it.

---

## The divergence ledger

One document listing every way your tool differs from the bundle. Each entry:
an id, the difference, evidence (bundle file and line), a class, and **what
would settle it**.

- **LOST** — the bundle did this, your tool does not
- **INVENTED** — your tool does this, the bundle did not
- **UNKNOWN** — depends on whether the server changed

A worked example with the awkward cases is in
[examples/divergence-ledger.md](examples/divergence-ledger.md).

Three rules:

**An entry closes on evidence about itself**, never as a side effect of
another change. Write the evidence down.

**Closing means deleting the row.** A ledger holding thirty closed entries and
four open ones hides the four. Closed entries live in git history.

**When you close one, grep for references to its id.** Code comments outlive
the entries they point at. Replace "see D-3" with the fact itself.

---

## Settle UNKNOWN by asking the server

Most UNKNOWN entries look unanswerable from the two repos. They usually are.
They are also usually trivial to answer by asking the running server.

In one review, four entries recorded as unsettleable were resolved in minutes:

- **Three fields the bundle always sent** — the server's OpenAPI
  (`/api/.../openapi.json` on a FastAPI backend) showed its request body no
  longer has them. Nothing was lost; the server moved on. Closed.
- **A `view=raw|effective` parameter the port omitted** — the server's `info`
  endpoint still advertised `views: ['raw','effective']` and both worked. A
  real loss. Restored.
- **A bound the bundle enforced client-side** — sending an out-of-range value
  returned HTTP 422 naming the limit. Real, and worth checking locally to save
  a round trip.
- **Vocabulary that looked invented** — declared in the server's own OpenAPI
  parameter list. Not ours.

**Order of ground truth:** the server's OpenAPI, then its `info`-style
self-description, then its error messages. All three outrank the bundle's
documentation and yours.

---

## Test claims about the bundle, do not assert them

Statements about what the bundle does are claims about behaviour. Several
made confidently in one conversion were false on first contact — including a
documented two-call procedure (`get` the member, then `graph` for
reverse edges) for something that takes one call, because the member record
already carried the field.

The same applies to examples carried from the bundle's documentation. **Run
every one.** Most of the examples inherited from one bundle's agent
description named resource types (`projects`, `initiatives`, docs) the server
no longer had — `info` reported only two types. They read as authoritative and
a model follows them literally.

---

## Match the cost model

Porting prose faithfully while changing *when* it loads is a real regression,
and invisible, because loading conditions are not in the content.

The bundle typically has zero always-on context: its reference doc loaded only
when the mode activated or the agent spawned. If your equivalent loads on
every call, that is a cost regression. If something the bundle always had now
loads sometimes, that is a correctness regression.

For each thing carried across, write down what triggered it in the bundle and
what triggers it now.

---

## Do not say "parity"

Not while a behavioural difference is open. If the bundle's agent could call
`submit_answer` and your model loop cannot — or can only behind a flag the
bundle never had — that is not parity, whatever the capability counts say.

Either match it or record the divergence.

---

## Done when

- Every inventory item is ported, deliberately dropped with a reason, or
  listed as having no counterpart.
- The ledger has no open entries, or the open ones are explicit accepted
  decisions.
- No comment or doc refers to a ledger id that no longer exists.
- Every capability has run against the real server — including the
  model-backed one, which is the one people skip.
- Every example in your `SMART_TOOL.md` has been executed.
