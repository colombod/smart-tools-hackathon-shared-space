# How the conversion goes wrong

Not a bug list. These are the process failures — wrong source, wrong
assumption, invented feature, premature "done" — that cost the most time
converting a bundle into a smart tool.

---

## Reading the wrong thing

**There are two spec repos.** One is a predecessor. They disagree, and the
older one is missing files the newer one has. Work proceeded against the
smaller one for a while, which made some of the documentation not merely
unclear but non-conformant.

Check which has the complete file set, and fetch before reading — a checkout
a few commits behind gives you answers that were true last month.

**There are two copies of the bundle.** The repo you cloned, and
`~/.amplifier/cache/<bundle>-<hash>/`. They can differ. Statements about "what
the bundle does" made from the wrong one are worthless and hard to detect.
Name the path you are reading before making any claim about bundle behaviour.

**The spec has gaps, and they are not your fault.** Ours defined an
`ecosystem`/`runner` vocabulary that had no home in any file. When something
in the spec does not resolve, say so and pick a defensible answer, rather than
assuming you have misread it.

---

## Assuming instead of checking

Every one of these was stated confidently and was wrong.

**"The bundle needs two calls for this."** The documented `get` the member,
then `graph` for reverse edges — the member record already carried the field.
One call.

**"The server has archived items, so this filter matters."** It had none.

**"These provenance fields were lost in the port."** The server's OpenAPI no
longer accepts them. Nothing was lost; the server moved on.

**"This cannot be settled from these two repos."** Four divergences were
recorded that way. All four were answered in minutes by reading the server's
OpenAPI and calling its `info` endpoint.

**The rule:** a claim about behaviour is settled by running something. Order
of ground truth: the server's OpenAPI, then its `info`-style self-description,
then its error messages. All three outrank the bundle's documentation and
yours.

**Bundle examples are claims too.** An agent description carried six worked
examples; most named resource types the server no longer had, and a model
follows them literally. Execute every example you carry across.

---

## Two different models, one word

A smart tool with a model-backed capability has **two** models in play, and
they get confused constantly — in documentation, in design arguments, and in
reasoning about what is allowed.

**The inside model.** The one your model-backed capability drives, in its own
loop, paid for with the caller's provider key. You choose which tool schemas
it receives.

**The outside model.** Whatever calls your tool — an Amplifier agent, a host,
a script. You do not control it, you do not choose its tools, and it can call
every capability you ship.

Saying "the model" without specifying which produces statements that are
false. "We withheld `configure` from the model" reads as a permission rule and
is not one: the inside loop is not handed that schema, while any outside
caller runs `configure` freely. One sentence, two meanings, and the wrong one
is the one a reader takes.

The confusion also corrupts design. A confirmation flag looked like a safety
gate because it was imagined against the outside model; against the inside
loop, which set the flag itself, it gated nothing. See "Building guards that
guard nothing".

**Check:** every time you write "the model", say which one. If the sentence is
about tool schemas, it is the inside loop. If it is about who may call your
tool at all, the answer is everyone, and no wording in your catalogue changes
that.

---

## Inventing things the bundle never had

This is where most of the time went. Everything below was built, shipped, and
later deleted — none of it had a counterpart in the bundle.

On the model-backed capability: a step budget, a hand-parsed action protocol
instead of native tool-calling, a null "I decline" outcome, validation of the
model's citations against what it had fetched, and a response size cap.

On configuration: an auth-mode environment variable with deprecation
machinery, an override to pin the auth choice, a second config file format, a
second config-resolution entry point, and an optional packaging extra that no
code branched on.

On the surface: two capabilities the bundle never had (an identity probe and a
bulk download — the latter the only path that wrote to local disk), a
confirmation flag on a capability that only writes the caller's own settings,
a permission flag gating the model's writes, and seven fields on a result
object where three were permanently null.

And the largest: an entire Amplifier adapter layer, which the spec does not
require and no authoritative example ships.

**The test, before adding anything:** does removing it change any answer a
caller can observe? Run it both ways. "The server supports it" is not a
reason. "It came with the code I was handed" is not a reason.

**The compound version:** a hand-rolled protocol produced parse failures,
retries, a step budget, an invented refusal outcome and citation validation —
each investigated as its own bug. All of them vanished when native tool-calling
replaced it. When defects cluster, suspect the foundation.

---

## Confusing the mechanism with what it delivered

A bundle's wiring — the mode, `contributes.agents`, `contributes.context`,
`tools.safe`, `model_role`, `includes:` — has no counterpart in a library and
CLI. That much is fine and should be recorded once.

The trap is concluding that because the mechanism cannot port, the thing is
resolved. Each of those mechanisms usually *delivered* something: the mode
delivered standing-order prose, the agent delivered an instruction document,
`contributes.context` delivered a reference.

We recorded "no behavioural guidance reaches the caller", then closed it by
deleting the adapter that carried the guidance — which made the statement
completely true rather than partly true. Three attempts to get right.

**Check:** state the problem without naming the thing you removed. If it still
describes something true, it is not solved.

**The related one:** porting prose faithfully while changing *when* it loads.
A bundle's reference document loads only on mode activation or agent spawn —
zero always-on cost. If yours loads on every call, that is a regression the
content cannot reveal. Write down what triggered each piece in the bundle and
what triggers it now.

---

## Building guards that guard nothing

We gated a write behind `confirmed=True`. The model loop passed the flag
automatically, so the caller most likely to write without asking never met the
gate. Every other caller simply set it. A flag cannot tell who set it — our
own documentation said so, below the paragraph presenting it as the
safeguard. And the bundle had no such flag.

Removed rather than hardened. The capability description is the guard: it
states that nothing can stop the call and that an agent must get approval
first.

**Check:** name the caller your gate stops. If they could remove it by passing
an argument, write documentation instead. Real enforcement belongs in
Amplifier — a mode's tool policy, or a human in the loop. A library cannot
gate its own caller.

Related: keep `destructive` meaning one thing. Ours drifted onto `configure`,
which writes the caller's own settings file, so the CLI announced that setting
your endpoint "writes to shared data".

---

## Closing things that are not closed

**A divergence closes on evidence about itself**, never as a side effect of
another change. Write the evidence down. If you cannot, it is not closed.

**A closed entry is deleted, not annotated.** A ledger holding thirty closed
rows and four open ones hides the four.

**Grep for the id when you close one.** Code comments outlive the entries they
point at; replace "see D-3" with the fact itself.

**Do not say "parity"** while any behavioural difference is open. Claiming it
once with a known difference — a permission flag the bundle never had — costs
more trust than the difference itself.

---

## Calling it done too early

**`status` must run with nothing configured and never raise.** It is the
conformance smoke check. Ours raised, because it called a helper catching only
one narrow exception type.

**A skipped conformance rule is not a passed rule.** Find out why it skipped;
ours skipped every runtime rule on the first run because the tool was not
installed.

**Every capability must run against the real server before you claim
completion** — including the model-backed one, which is the one people skip
because it costs money and is slow. That is also the one most likely to be
broken.

**A caveat means not done.** "Finished, except the model-backed path has never
run end to end" is not finished. In our case the blocker was that two provider
keys were set, the tool silently preferred one, and that one was invalid —
which was worth knowing on its own.
