# The catalogue

The spec does not mandate this. Skip it and you will spend your time on names
disagreeing across surfaces instead of on the tool.

A smart tool states the same fact in five places: the library function, the
CLI verb, the JSON schema a model receives, the `SMART_TOOL.md` capability
list, and human documentation. Write it five times and you have five things
that drift. Nothing fails; the surfaces simply stop agreeing.

Declare each capability **once**, in data. Derive everything else.

---

## What a catalogue entry holds

```python
Capability(
    verb="get",
    description="...",              # verbatim from the bundle, if porting
    returns="dict -- the record",
    arguments=(Argument(name="id", type="str", required=True, help="..."),),
    destructive=False,
    model_backed=False,
    in_model_loop=True,
    withheld_because="",            # required when in_model_loop is False
)
```

From that, generate: the CLI parser, the model-facing tool schemas, and the
manifest's capability list. The library function is written by hand but its
name and parameters are asserted against the entry by a test.

**If generating the CLI is awkward, the catalogue is missing something.** Fix
the catalogue, not the generator.

---

## The rules that actually matter

### One name, everywhere

`get` in the library, `get` in the schema, `--id` matching `id`. The only
permitted transformation is underscores becoming hyphens on the CLI, done at
that one boundary.

If you find yourself writing a mapping so two names can coexist — an argparse
`dest=`, a rendering function, a prefix rule — delete the mapping and pick one
name. A conversion built a per-surface name renderer before someone asked why;
making the names identical broke nothing.

A capability named after an ordinary English word (`get`, `status`, `search`)
is fine. If a sentence elsewhere reads ambiguously, quote the word —
`` `get` `` — rather than renaming across five surfaces.

### Identity, not equality

Assert that the description a model receives **is the same object** the
library reports, not two strings that compare equal:

```python
assert capability.as_tool()["description"] is capability.description
```

Equality passes right up until someone edits one copy. Identity makes drift
impossible rather than currently absent.

### Copy bundle descriptions verbatim

If you are porting, the bundle's tool descriptions and JSON schemas are the
specification. Do not rewrite them in your own words. Where a description must
deviate — the bundle's example names a resource type the server dropped — mark
it inline:

```python
# DEVIATION: the bundle's example is `projects/x`; the server no longer has
# that type. `info` reports the authoritative list.
```

Inline, not in a separate document. A separate list goes stale, and the person
reading the description will not know to look for it.

---

## Three ways derived surfaces still break

All three shipped.

**Truncation in a derived view.** A CLI listing took the first paragraph of
each description. The one capability whose description had two paragraphs was
the one that writes, and the dropped half was its safety text.

Compare the derived view against the source for *every* entry, not a sample.

**A `to_dict` that forgets a field.** `Manifest` carried `body`; `to_dict()`
did not list it. `manifest` returned frontmatter only, so every consumer
reading it to learn how to use the tool got the summary and none of the
guidance. Nothing errored.

Test that every dataclass field appears in its serialized form. One test,
written once, catches the class forever.

**A schema that drifts from the entry.** Assert that the generated schema's
property names and required list match the catalogue's arguments, for every
capability.

---

## Which capabilities the model loop may call

**This is about one specific model: the one your model-backed capability
drives.** It is not a permission system. Every capability remains callable by
anyone — the library, the CLI, an agent that shells out to your tool. The only
question here is which tool schemas you hand to your own loop.

Some do not belong there: `configure` (the loop does not know the deployment
URL, and a guess writes a wrong value to disk), `status` (a diagnostic the loop
cannot act on), and the model-backed capability itself (it would call itself).

**Record the reason in the entry.** A capability quietly missing from the
loop's schemas is indistinguishable from an oversight, and gets re-argued
months later. A test that fails when `withheld_because` is empty turns that
into a decision.

Do not leave writes out reflexively. If the bundle's agent could write, your
loop should too — see [converting-a-bundle.md](converting-a-bundle.md).

---

## The tests that keep it honest

Write these against your catalogue. Each one caught a defect that had already
shipped:

| Test | Catches |
|---|---|
| Description identity | Two copies drifting |
| Argument names match library signature | A translation hiding an inconsistency |
| Every dataclass field survives `to_dict` | A serializer silently dropping one |
| Schema matches entry for every capability | Generated output drifting |
| Truncated help is a prefix of the full text | Losing the tail of the longest entry |
| Withheld capabilities have a reason | Decisions degrading into accidents |
| Model-backed capabilities say they cost tokens | A caller unable to weigh cost |

Prose does not prevent behaviour. Several rules in these documents were
violated by the person who wrote them, within the same session. A failing test
does not compete for attention; it blocks.
