# Smart tools: what the spec doesn't tell you

You have the smart-tool spec. It tells you what files to produce and what
shape they take. These notes cover what it leaves out, written from one
conversion of an Amplifier bundle into a standalone smart tool.

Assumes you are a competent developer with the spec open. Nothing here repeats
the spec, and nothing here is general software advice.

**One piece of vocabulary, because everything else leans on it.** If your tool
has a model-backed capability, two models are in play: the *inside loop* your
tool drives with the caller's provider key, and the *outside caller* — an
agent or host invoking your tool. "The model" means nothing on its own, and
conflating them produces false statements about what is allowed. See
[traps.md](traps.md), "Two different models, one word".

| | |
|---|---|
| [converting-a-bundle.md](converting-a-bundle.md) | What in a bundle maps to what in a smart tool, what has no counterpart, and how to know when you are done |
| [the-catalogue.md](the-catalogue.md) | Declaring each capability once and deriving every surface from it — not spec-mandated, and the highest-leverage decision in the build |
| [decisions-the-spec-leaves-open.md](decisions-the-spec-leaves-open.md) | Model exposure, write guards, config location, auth selection, install-time setup — the spec is silent, so these are yours |
| [traps.md](traps.md) | How the conversion goes wrong — wrong source, wrong assumption, invented feature, premature "done" |
| [CHECKLIST.md](CHECKLIST.md) | The sequence, with tick-boxes |
| [examples/divergence-ledger.md](examples/divergence-ledger.md) | A worked ledger, including the awkward entries |

---

## The three that carried the most weight

**Ask the server, not the repos.** Four divergences recorded as
"unsettleable" were all closed in minutes by reading the server's OpenAPI and
calling its `info` endpoint. One turned out to be a real capability loss and
was restored; three were the server having moved on. Speculation had cost far
more than the commands did.

**Does removing it change any answer?** Two filter parameters survived long
debate because the server documented them. Running with and without showed the
server had one value to filter by and nothing archived — neither could change
a result. Both deleted. Most over-building dies to this question, asked as a
command rather than an argument.

**A caveat means not done.** "Finished, except the model-backed path has never
run end to end" is not finished. That path is the least-exercised and the most
likely to be broken — in our case by a retired model name that a full green
test suite never noticed.
