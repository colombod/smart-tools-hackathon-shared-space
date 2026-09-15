# Checklist

Tick a step only when you have output to show for it. Each links to the
detail.

## Before writing code

- [ ] Confirm which spec repo is authoritative — more than one exists, and one
      is a predecessor. Fetch before reading; a stale checkout gives answers
      that were true last month.
- [ ] Confirm which copy of the bundle you are reading — the repo you cloned,
      or `~/.amplifier/cache/<bundle>-<hash>/`. They can differ.
- [ ] Inventory the bundle file by file, with line counts. The largest file is
      usually prose, and usually the most valuable thing in it.
- [ ] Mark each piece: content (ports) or wiring (does not). For each wiring
      piece, note what it *delivered* — that payload still has to arrive.

→ [converting-a-bundle.md](converting-a-bundle.md)

## Catalogue first

- [ ] Declare every capability in one place before writing any of them.
- [ ] Copy the bundle's tool descriptions and schemas verbatim; mark each
      deviation inline with the fact forcing it.
- [ ] Generate the CLI and the model schemas from it. If that is awkward, the
      catalogue is missing something.
- [ ] Record a reason for every capability your model loop does not get.

→ [the-catalogue.md](the-catalogue.md)

## Decide deliberately

- [ ] Which capabilities your model loop may call — the spec says nothing.
      (Only your own loop; every capability stays callable by anyone else.)
- [ ] How writes are guarded — `destructive` is not in the spec, and a
      confirmation flag the caller can set guards nothing.
- [ ] Where config lives, and the precedence order.
- [ ] How auth is selected, and what a malformed credential does.
- [ ] Confirm nothing deployment-specific is hardcoded, including inside error
      message text.

→ [decisions-the-spec-leaves-open.md](decisions-the-spec-leaves-open.md)

## Before adding anything

- [ ] Does removing it change any answer a caller can observe? Run it both
      ways.
- [ ] "The server supports it" and "it came with the code" are not reasons.
- [ ] If it translates between two names, try making the names identical
      first.

→ [traps.md](traps.md), "Inventing things the bundle never had"

## Ledger

- [ ] Every divergence has an id, evidence, a class, and what would settle it.
- [ ] Every UNKNOWN settled by asking the server — OpenAPI, then `info`, then
      error messages.
- [ ] Closed entries deleted from the open list, and their ids grepped for in
      code comments.
- [ ] You have not used the word "parity" for anything with a known
      difference.

→ [examples/divergence-ledger.md](examples/divergence-ledger.md)

## Before saying it works

- [ ] Every capability run against the real server — including the
      model-backed one.
- [ ] Every example in `SMART_TOOL.md` executed.
- [ ] Name which lines of your network and auth code have ever run. If every
      test uses a fake client, the answer is none.
- [ ] Conformance passes with no failures **and no skips** — a skip is not a
      pass.
- [ ] Wheel builds; open it and confirm the contents.
- [ ] **Any caveat means not done.**

→ [traps.md](traps.md)
