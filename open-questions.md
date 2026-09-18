# What we did not settle

The other files say what we found. This one says what we **still do not know**, what it
would take to find out, and which of it is worth anyone's next day.

Ordered by how cheaply a real answer could be had, not by how interesting the question is.

---

## Ready to test tomorrow

### 1. Does an intent-shaped description actually get picked up?

**Status: FIXED, NOT MEASURED.** This is the honest gap in our loudest finding.

All three of our tools shipped skill descriptions that matched the tool's **name** rather
than the user's intent, and a real user in Codex had to name the tool explicitly every
time. We rewrote all three to open on quoted user phrasings. We have **not** verified that
the rewrite works.

**What would settle it:** the same harness, the same task, phrased without the tool's name
— *"trim this down to where she explains the pricing"*. If it still needs prompting, the
matcher wants something structural and better prose will never fix it. **A negative result
here matters more than the rewrite did.**

### 2. Can a conformance kit check *reachability*?

Two of our tools passed 15/15 while carrying a description no user phrasing could reach,
and `use_cases` that was a single sentence copy-pasted from `description`.

**Two candidate checks, both lints rather than judgment calls:**

- `use_cases` must not duplicate `description`
- warn on a description whose only hook is a keyword list or the tool's own name

**What would settle it:** implement both against the catalog and count how many existing
tools trip them. If it is most of them, the checks are right and the convention is missing.

### 3. Does the "deterministic check" rule generalise?

Our strongest claim: **a model-backed capability is safe to ship unattended when a cheap
deterministic check exists on its output.** Three instances in one tool — a generated
ffmpeg expression sampled mid-blend, a narration line measured against its slot, a colour
transfer scored by statistical distance.

Three instances in **one tool by one author** is not a law.

**What would settle it:** take three model-backed capabilities from tools nobody here
wrote, and ask of each — is there a cheap check on the output, and does the tool run it? If
the ones with checks are the ones people trust, the rule holds.

---

## Needs a schema change, so needs agreement first

### 4. `model_backed` is a boolean; real tools are plural and tiered

One of our tools uses **four kinds of intelligence**: speech-to-text and text-to-speech
**locally with no credential**, plus text reasoning and vision remotely. A consumer asking
*"what AI does this use?"* gets a four-row table, and two rows need no credential at all.

`requires[]` cannot say **which capability each entry unlocks**. The only place that mapping
exists in our tools is prose we wrote by hand.

**What would settle it:** a proposed shape, tried against a tool that has exactly one model
capability and one that has four. If the same schema serves both without ceremony, it is
right.

### 5. `requires[]` names a binary and cannot describe it

A real macOS install found all three of these missing at once:

- **which build features** of a binary you need — `ffmpeg` is necessary and not sufficient;
  `caption` needs it built with libass
- **that an install may need a postinstall step** — `brew postinstall ca-certificates
  fontconfig gnutls glib openssl@3`, or captions render with no text in them
- **that only one capability breaks** without it — every other verb was fine

All three were true, none expressible, so the tool carries them in prose.

### 6. Should the contract say vision is already possible?

`AgentRequest` has no image field, so the natural conclusion is that the seam cannot do
vision. **It can** — the request carries a `workspace`, and the agent behind it has
file-reading tools. We verified it before building on it.

A builder reaching the natural conclusion would fork the seam, add a field, or go straight
to a provider SDK — all of which break the boundary the scaffold exists to create.

**What would settle it:** one sentence in the interface docstring. The question is only
whether anyone else has already hit this and solved it differently.

---

## ROADMAP questions we did not touch

Stated plainly so nobody assumes they were covered.

| # | question | our position |
|---|---|---|
| 5 | **Continuing a smart call** — clarification, session continuation | **No evidence either way.** We never needed it. That is weak evidence it is unnecessary, and we may simply have built tools that do not want it. |
| 6 | **Generated wrappers** — an SDK or MCP wrapper over an existing tool | **Untouched.** A generator over one of ours would be a day's work and would test whether the manifest carries enough to generate from. |

On **#1 (long-running)** we have a position and an implementation — `--detach`/`--wait`,
with in-flight runs deliberately not durable past the calling session. Whether those should
be **standard verbs in the spec** is still open, and is a decision for the spec's authors
rather than a thing we can settle by building.

---

## Things we would explore if the hackathon ran another week

**A tool nobody here wrote, taken end to end.** Every finding in this repo comes from tools
written by their own finder. The failure modes we cannot see are the ones that only appear
when the author is not in the room.

**Cross-platform, seriously.** Our DTU proved the install works on Linux and told us
nothing about macOS — a user found the real gap in about a minute. A container per platform
is cheap; we did not do it, and it is the single highest-yield thing on this list.

**Whether a smart tool should ship evaluation at all.** We built one and it stayed thin.
Open: is a per-tool eval harness the author's job, the kit's job, or nobody's?

**The negative results are worth as much as the positives.** Three from this build, all
documented in the findings and all worth repeating deliberately:

- a "works without a provider" test that passed because the environment leaked a credential
  store
- a no-ffmpeg suite that ran **with** ffmpeg, because the scrub was a guess about which
  system directories were safe
- a flat-grey fixture that reported two *opposite* colour grades as byte-identical

Each was a check that passed for a reason other than the property holding. If there is one
convention worth arguing for beyond the spec, it is that **a negative test must prove the
absence it depends on** — and none of our three did until something outside our machine
said so.
