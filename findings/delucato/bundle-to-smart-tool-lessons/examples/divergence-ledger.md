# Example: a divergence ledger

This is a worked example of the ledger described in
[../converting-a-bundle.md](../converting-a-bundle.md). The entries are real
ones from a bundle conversion, lightly anonymised, including the awkward
cases.

Copy the shape. The value is in the columns, especially the last one.

---

## What this document is for

One place listing every way the smart tool differs from the bundle it came
from, so the question "are we done?" has an answer.

**Keep only what is open.** Closed entries live in version control history.
A ledger with thirty closed rows and four open ones hides the four.

Each entry is classified:

- **LOST** — the bundle did this and the smart tool does not.
- **INVENTED** — the smart tool does this and the bundle did not.
- **UNKNOWN** — cannot tell yet; usually depends on whether the server
  changed.

---

## Open

| Id | Divergence | Evidence | Class |
|---|---|---|---|
| D-2 | **Error envelope reshaped.** The bundle passed the server's `{code, message, status}` through verbatim. The port emits `{type, message}`, so a caller cannot branch on the server's code without parsing JSON out of a string. | bundle `modules/tool-x/client.py:95-116` | INVENTED |
| D-5 | **No logging.** The bundle logged one line at `mount()`. | bundle `modules/tool-x/tool.py:495` | LOST |
| D-9 | **Bulk export capability absent.** The bundle could write the whole corpus to local disk. The port has no capability that writes to local disk at all. | bundle `modules/tool-x/tool.py:212-240` | LOST |

## What would settle each

**D-2** is a judgement call needing no new information. Three options: leave it
and document that the server's code is inside the message; add the server's
fields alongside ours; nest the server's envelope under its own key. The third
keeps provenance explicit and costs one nested key.

**D-5** is a judgement call. Note what the bundle actually logged: one line at
`mount()`, when Amplifier loaded the module. A library and a CLI are never
mounted — so the one thing the bundle logged is the one thing that cannot
happen here. Adding request logging would be an invention, not a port.

**D-9** needs a decision from whoever owns the product. The capability was
real and worked. It was also the only path that wrote to local disk, which is
a large thing for a read-oriented tool to carry. Either port it deliberately
or drop it deliberately; do not leave it here.

---

## Accepted — differences kept, with reasons

These are decisions, not open questions. They live here rather than in history
because someone will ask about each one again.

| Id | Divergence | Reason |
|---|---|---|
| A-1 | A CLI exists at all. The bundle ran only inside an Amplifier session. | Required by the spec. |
| A-2 | `status` and `configure`, which the bundle had no equivalent of. | Required by the spec. The bundle relied on Amplifier for both. |
| A-3 | Settings live in the tool's own `.env` rather than Amplifier's `settings.yaml`. | Forced by standalone operation: there is no Amplifier `settings.yaml` to read. |
| A-4 | Failure at call time, not at import. | The bundle failed fast at `mount()` to avoid repeated auth errors. A library must not do credential work at import; `status` provides the same early warning. |
| A-5 | A model-backed capability, which the bundle did not have. | Accepted addition. The bundle got this by delegating to its expert agent; with no Amplifier session to delegate into, the tool provides it directly. This is the tool's reason to exist. |

---

## Structurally not portable

Recorded once so the question is closed rather than rediscovered.

| Piece | Why it cannot come across |
|---|---|
| The mode (`/name`, `shortcut`) | A library is called and returns; a CLI process exits. No session to overlay. **Its prose was ported** into the manifest body. |
| `tools.safe`, `default_action` | Amplifier decides what may be called. By the time this code runs, that decision is made. |
| `contributes.agents` | Delegation forks a child session. **The agent's instruction was ported** as the model-backed capability's system prompt. |
| `contributes.context` | Load-only-while-active has no counterpart. **The document was ported** into the manifest body, where it always loads. |
| `model_role` | A hint to Amplifier's router. No counterpart. |
| `includes:` in `bundle.md` | A wheel declares dependencies; nothing composes behaviour that way. |

Note the right-hand column: most of these carried a payload, and the payload
ports even though the mechanism does not. Concluding "the mechanism cannot
port, therefore this is resolved" is how content gets lost. See "Confusing
the mechanism with what it delivered" in [../traps.md](../traps.md).

---

## Closed — for illustration only

A real ledger would not keep this section; these rows would be deleted and
live in version control history. They are shown here to demonstrate what
closure looks like, and specifically that **an entry closes on evidence
specific to itself.**

| Id | Was | Closed because |
|---|---|---|
| D-1 | **Three provenance fields dropped from a write.** The bundle always sent them. UNKNOWN. | The server's OpenAPI lists the accepted request body. None of the three exist any more. Nothing was lost — the server changed, and the port matches its current schema. |
| D-3 | **A `view` parameter omitted.** The bundle exposed raw vs effective. UNKNOWN. | The server's `info` still advertises both views and both work when called. A genuine loss. **Restored.** |
| D-4 | **A bound not enforced.** The bundle checked a limit client-side. LOST. | The server returns HTTP 422 naming the limit, so nothing was corrupted — but a caller spent a round trip to learn it. **Bound now checked before the request.** |
| D-6 | **Two filter arguments the bundle never had.** INVENTED. | Run both ways: the server has exactly one value to filter by, so filtering narrowed nothing, and nothing was archived, so the second returned nothing. Neither could change any answer. **Removed.** |
| D-7 | **No behavioural guidance reaches the caller.** The bundle's mode shipped prose telling the assistant when to reach for it. LOST. | Ported into the manifest body, which any consumer reads. **Note:** wrongly closed once as a side effect of deleting the adapter that carried the prose — which made the gap total rather than fixing it. Reopened, then closed properly. |

All but the last were settled by **asking the running server** — its OpenAPI
and its `info` endpoint — rather than reasoning from the two repos. That is
the highest-value move in a conversion. See
[../converting-a-bundle.md](../converting-a-bundle.md).

---

## Rules this example demonstrates

- Every entry carries evidence — a file and line in the bundle.
- Every open entry says **what would settle it**, not merely that it needs a
  decision.
- UNKNOWN entries are settled empirically, not by argument.
- An entry closes only on evidence about itself. D-7 shows what happens
  otherwise.
- Accepted differences are recorded with reasons, because they will be
  questioned again.
- Things that cannot port are listed once, with a note on where their content
  went.
