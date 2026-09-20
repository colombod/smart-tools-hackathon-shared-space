# Output-correctness sweep — `deep-research` / `fact-check` 0.9.1

Investigation only. **No file inside any repository was modified.** Everything here was
produced on 2026-09-20 against the installed tools on PATH; raw outputs are in `raw/`.

| | |
|---|---|
| tools | `deep-research` 0.9.1, `fact-check` 0.9.1 (`raw/01-check.txt`) |
| host | `ready: true` — perplexity + anthropic both satisfied; openai/gemini credential present but client library absent |
| runs created | 4 research, 3 fact-check (`raw/38-cost-ledger.txt`) |
| **cost reported by the tools** | **$0.1896** |
| **cost actually spent** | **$0.5336** (see D1 — the tools under-report by 2.8x) |

## Verdict at a glance

| # | Question | Verdict |
|---|---|---|
| 1 | Does a brief rest on the sources it cites? | **PARTLY.** The answer is factually correct, but the *primary* citation — the one the brief names as the reason its confidence is "high" — is a **404**. Nothing in the tool ever checks that a cited URL resolves. |
| 2 | Do fact-check's verdicts discriminate? | **YES, 4/4 correct.** supported / refuted / unverifiable / opinion, all with justified reasoning. No bucket-swallowing. But the structured `tally` field the docs call "the answer" is **always `null`** on a real run. |
| 3 | Does `deep-research` → `fact-check` compose? | **MECHANICALLY YES** (ids, URLs and `inherited_from` transfer exactly, no dangling citations) — **but two seam defects break `--detach`**, and the composition faithfully propagates the dead citation into four "high"-confidence verdicts. |
| 4 | Does a detached run report truthfully, including on failure? | **MOSTLY YES** — the regressions named in the brief are genuinely fixed. Three new honesty gaps found: cost under-reporting on the failure path, and two preflight checks that the detached path skips. |

## Defects found (all reproduced, all with evidence)

| id | what | severity |
|---|---|---|
| **D1** | A run that fails `attempts_exhausted` reports `attempts_discarded: 0`, `discarded_cost_usd: null` and 1/13th of its real cost — in the same document that says "rejected 3 times" | high (silent money) |
| **D2** | `fact-check verdicts` always returns `tally: null`; reader keys on `tally`, writer writes `counts` | high (documented headline answer is dead) |
| **D3** | A relative `--claims-file` works attached and **always** fails detached — child is spawned with `cwd=runs_dir` and handed the path verbatim | high (detach-only breakage) |
| **D4** | `--detach` accepts a `--from-run` that does not exist, and reports it as `inherited_from` | medium (accepts what attached refuses) |
| **D5** | No citation liveness check anywhere: a 404 URL is published in the brief, the sources view and the bibliography, unmarked | medium (correctness-of-answer) |
| **D6** | Repair feedback misdiagnoses a malformed-JSON reply as "no JSON document", burning the whole attempt budget on the wrong instruction | medium (wasted spend) |
| **D7** | Every error envelope goes to **stderr** while both tools' docs say the JSON document — success *or* failure — is on **stdout** | low (parser contract) |
| **D8** | Documented "a refusal carries `affordances` too" — no refusal observed carried them | low (doc vs behaviour) |

---

# Question 1 — Does a brief rest on the sources it cites?

## What I ran

A question with an answer checkable against a primary source, deliberately not contested
and not fast-moving:

```bash
deep-research research \
  --query 'What are the maximum lengths, in octets, of a single DNS label and of a complete domain name, as stated in RFC 1035?' \
  --depth low --max-sources 10 --detach
```

→ `dr-6f447489`, complete in 34.5 s, 9 sources, reported $0.054127, no discarded attempts
(`raw/12-q1b-launch.json`, `raw/13-q1b-status.json`, `raw/30-q4-healthy-events.txt`).

An earlier attempt on a licence question (`dr-dd376b7d`) failed — analysed under D1/D6 below.

## The brief (`raw/14-q1b-brief.json`, verbatim)

> RFC 1035 sets the maximum DNS label length at 63 octets and the maximum complete domain
> name length at 255 octets (wire-format, including length-prefix octets and the root
> label) **[s1]**. Confidence is high: **the actual RFC 1035 text is in the source list [s1]**,
> with a mirror of its Section 3.1 definitions also available **[s4]**. No conflicting
> figures were found in the evidence provided for the RFC's own stated limits.

## Is the answer right?

**Yes.** Verified against the real primary source, fetched independently
(`raw/20-rfc1035.txt`, 122 549 bytes from `https://www.rfc-editor.org/rfc/rfc1035.txt`):

```
2.3.4. Size limits
labels          63 octets or less        <- line 523
names           255 octets or less       <- line 525
```

and §3.1 line 544: *"the total length of a domain name (i.e., label octets and label length
octets) is restricted to 255 octets or less"*. The brief's wire-format caveat is correct too.

## Per-citation audit (`raw/18-resolve.txt`, `raw/19-s1-probe.txt`, `raw/21-s4-freesoft.html`)

| id | URL | resolves? | does it say what is attributed to it? |
|---|---|---|---|
| **s1** | `rfc-editor.org/rfc/pdfrfc/rfc1035.txt.pdf` | **404 — NOT FOUND** | **cannot: the page does not exist** |
| s4 | `freesoft.org/CIE/RFC/1035/11.htm` | 200 | **SUPPORTED** — verbatim RFC 1035 §3.1, contains both "63 octets or less" and "255 octets or less" |
| s2 | `devblogs.microsoft.com/oldnewthing/20120412-00/?p=7873` | 200 | n/a — cited by title only, report explicitly disclaims its content |
| s3 | `spec.school/topics/dns/lessons/names-and-labels` | 200 (1 330 B JS shell, canonical → site root) | n/a — title only |
| s5 | `tex2e.github.io/rfc-translater/html/rfc9267.html` | 200 | n/a — not cited; irrelevant to the question (RFC 9267) |
| s6 | `metacpan.org/...Regexp/Common/dns.pm` | 402 | n/a — title only; bot-blocked, not proven dead |
| s7 | `mhonarc.org/archive/html/spf-discuss/2004-11/msg01329.html` | 403 | n/a — title only; bot-blocked |
| s8 | `learn.microsoft.com/...naming-conventions...` | 200 | n/a — not cited |
| s9 | `groups.google.com/g/comp.protocols.dns.bind/c/8Lep8lp5TBU` | 429 | n/a — title only; rate-limited |

The s1 404 is not a transient or a bot-block — it is a real miss, and the neighbouring
paths were probed to prove it (`raw/19-s1-probe.txt`):

```
https://www.rfc-editor.org/rfc/pdfrfc/rfc1035.txt.pdf  -> HTTP/2 404, body "404 - Not found"
https://www.rfc-editor.org/rfc/rfc1035.txt             -> 200, 122549 bytes
https://www.rfc-editor.org/rfc/rfc1035.pdf             -> 404
https://www.rfc-editor.org/rfc/pdfrfc/rfc1035.pdf      -> 404
```

## D5 — the finding

The dead URL was **returned by the Perplexity backend**, not invented by the tool — it is
present verbatim in the run's own `raw/gather-01.json`. So this is not a hallucination. It
is worse in one specific way: **nothing between the backend and the user ever checks it.**

- `grep` over `research-core/src` and both tools' `src` finds **no HTTP fetch of a source URL
  anywhere** — no liveness check exists (`raw/41-no-url-liveness-check.txt`: every match for
  `requests.get|urlopen|httpx|status_code|reachable` is prose inside a docstring or an error
  message, none is a call).
- The run's only structural citation check is `dangling_citations` (`runs.py:461`), which
  catches a model citing `[s7]` when the run has six sources. It says nothing about whether
  a source's URL is real.
- The brief then uses the *presence* of s1 in the source list as its stated reason for
  **"high" confidence** — a claim about evidence quality resting on an artefact that was
  never opened.
- `deep-research render --format bibliography` publishes the dead link to a human reader
  under a "Docs" heading, unmarked (`raw/37-brief-and-bibliography.txt`).

**A citation that resolves to a real page is not the same as a citation that supports the
sentence attached to it. Here it is one worse: the load-bearing citation resolves to
nothing at all, and the brief's confidence claim is built on it.**

Verdict per the question asked: **1 citation SUPPORTED (s4), 1 NOT FOUND (s1), 7 not
attributed any content** (the report is scrupulous about saying so: *"their actual content
was not provided here … titles alone are not evidence of what position they take"*).

---

# Question 2 — Do fact-check's verdicts discriminate?

## What I ran

One run, four claims of four different kinds, against the evidence `dr-6f447489` gathered
(`raw/22-claims.txt`):

```bash
fact-check check-claims --from-run dr-6f447489 \
  --claims-file "$PWD/raw/22-claims.txt" --detach
```

→ `fc-45813ee9`, complete in 38.0 s, $0.096276, 4 verdicts (`raw/31-fc2-status.json`).

## Result — 4 of 4 correct (`raw/32-fc2-verdicts.json`)

| # | claim | should be | **got** | sources justify it? |
|---|---|---|---|---|
| 0 | "RFC 1035 limits a single DNS label to 63 octets or less." | supported | **supported** (high) | yes — s4 genuinely says this; s1 would too if it resolved |
| 1 | "RFC 1035 limits a complete domain name to **512** octets or less." | refuted | **refuted** (high) | yes — and it caught *why* the number is plausible: *"512 octets is actually the traditional maximum size of a DNS UDP message … the claim appears to conflate two distinct RFC 1035 figures"* |
| 2 | "The authors … picked 63 octets because of a memory constraint in the PDP-10 machines at ISI in 1987." | unverifiable | **unverifiable** (high) | yes — *"the number itself is well-documented, but the specific causal/historical claim about why that number was picked is not supported or contradicted by anything"* |
| 3 | "The 255-octet limit … is too small for modern applications and ought to be raised." | opinion | **opinion** (high) | yes — separates the factual baseline from *"a value judgment about design tradeoffs … rather than a fact that evidence can settle"* |

**No bucket swallowed anything.** The tally on disk is exactly `{supported: 1, refuted: 1,
unverifiable: 1, opinion: 1}`, and the prose brief leads with it correctly:

> 4 claims assessed: 1 supported, 1 refuted, 1 unverifiable, 1 opinion (not a factual claim).

This is the strongest result in the sweep. Claim 1 in particular is a real discrimination:
the wrong number I planted (512) *is* an RFC 1035 figure, just a different one, and the
verdict named the conflation rather than merely disagreeing.

## D2 — but the machine-readable tally is always `null`

```
$ fact-check verdicts fc-45813ee9 | jq '.result.tally, .result.count'
null
4
```

Both documents promise otherwise:

- `fact-check --help`: *"The **`tally`** travels inline because it is small and **it IS the
  answer**"*
- `fact-check verdicts --help` Result: *"`run_id`, `count`, **`tally` (counts per verdict)**,
  `verdicts`"*

Root cause, one key (`raw/33-tally-probe.txt`, `raw/34-fixture-vs-real.txt`):

- reader: `packages/research-core/src/research_core/runs.py:445` — `"tally": run.record.get("tally")`
- writer: `tools/fact-check/src/fact_check/checking.py:472` — `writer.count(sources=…, claims=…, **counts)` → lands in `run.json["counts"]`

No real `run.json` has ever had a `tally` key. Verified on this run:

```
REAL RUN keys: [... 'counts', ... ]   real['tally']  = None
                                      real['counts'] = {claims:4, opinion:1, refuted:1, sources:9, supported:1, unverifiable:1}
```

### How it survived 335 green tests

`packages/research-core/tests/test_runs.py:173 test_verdicts_filter_and_carry_the_tally`
asserts `document["tally"]["refuted"] == 1` — against a **hand-written fixture**,
`packages/research-core/tests/fixtures/runs/fc-9f8e7d6c/run.json`, which carries a `tally`
key no engine has ever written. The fixture's shape is a fiction in both directions
(`raw/34-fixture-vs-real.txt`):

```
in fixture, not in any real run:  ['from_run', 'tally']
in every real run, not in fixture: ['confidence', 'detached', 'host', 'inherited_from', 'pid']
```

> ★ **Insight:** the fixture was built from the fields the reader consults, so it could only
> ever confirm the shape the reader already assumed — `runs.py:445` reads `tally`, the
> fixture has `tally`, green. The one artefact that would have disagreed is the thing the
> tool actually produces. Principle: a fixture must be a recording of a real run, or it
> tests the test.

---

# Question 3 — Does the composition work?

## The evidence seam: works

`raw/36-composition-check.txt`, machine-compared:

```
dr-6f447489: count=9  inherited_from= None
fc-45813ee9: count=9  inherited_from= dr-6f447489
id->url identical across the seam: True       (s1..s9, every one)
source ids cited by verdicts: ['s1', 's4']
dangling (cited but not in the inherited set): []
```

So: evidence is genuinely reused rather than re-gathered, the provenance link is recorded
on both the envelope and the `sources` view, and every `[sN]` a verdict rests on exists in
the inherited set. This is the part the sibling video tool got wrong and this one gets right.

## What the seam also carries: the dead citation

```
the 404 URL is still s1 on the fact-check side: True
verdicts resting on s1: [(0,'supported','high'), (1,'refuted','high'),
                         (2,'unverifiable','high'), (3,'opinion','high')]
```

All four verdicts rest on s1 at **high** confidence, and verdict 0's reasoning opens
*"[s1] is the primary text of RFC 1035 itself"* — an assertion about a page that 404s. The
composition is faithful; that is precisely why an unchecked citation launders itself into
four more "high"-confidence artefacts one hop downstream.

## D3 — a relative `--claims-file` always fails under `--detach`

Same command, same cwd, **only `--detach` differs** (`raw/27-ab-relative-claimsfile.txt`):

```
$ fact-check check-claims --claims-file raw/22-claims.txt --from-run dr-DOES-NOT-EXIST
{"error": {"code": "run_not_found", "message": "No run 'dr-DOES-NOT-EXIST' ..."}}     exit=1
        ^ the claims file read FINE — the complaint is about the run

$ fact-check check-claims --claims-file raw/22-claims.txt --from-run dr-DOES-NOT-EXIST --detach
{"result": {"accepted": true, "claim_count": 4, ...}}                                  exit=0
```

and the child then dies (`raw/26-fc-failed-status.json`, first occurrence on the real run
`fc-55d7c403`):

```json
"failure": {"code": "usage", "message": "No claims file at raw/22-claims.txt.",
            "remedy": "Point --claims-file at a file with one claim per line. Log retained at …/detached.log."}
```

Note `claim_count: 4` in the acceptance — **the parent read the file successfully, counted
the claims, and then handed the child a path the child cannot resolve.**

Mechanism, exact:
- `tools/fact-check/src/fact_check/checking.py:176` — `"claims_file": claims_file` passed verbatim
- `tools/fact-check/src/fact_check/checking.py:198` — `cwd=str(runs_dir)` — the child runs somewhere else
- `tools/fact-check/src/fact_check/checking.py:57` — `Path(claims_file).expanduser()` — expands `~`, never `resolve()`s

Absolute path works (that is how `fc-45813ee9` ran). The fix is one `.resolve()` before the
handoff; the finding is the class: **the parent validates in one cwd and the child works in
another.**

## D4 — `--detach` accepts a `--from-run` that does not exist

Same pair above: attached refuses `dr-DOES-NOT-EXIST` immediately with `run_not_found`
exit 1; detached returns `accepted: true` **and reports `"inherited_from": "dr-DOES-NOT-EXIST"`**
as if it were real provenance. `checking.py:335` carries the comment *"A `--detach` request
that will die on its own preflight…"* — the evidence-run existence check is not in that
preflight. This is the same shape as the regression the brief named as fixed for the
provider surface, still open for the evidence surface.

---

# Question 4 — Does a detached run report truthfully, including on failure?

## The named regressions are genuinely fixed

**No provider configured → refuses, does not accept** (`raw/06-q4a-nocred-detach.err`):

```bash
env -u ANTHROPIC_API_KEY -u ANTHROPIC_KEY -u OPENAI_API_KEY -u GOOGLE_API_KEY \
    -u PERPLEXITY_API_KEY -u GITHUB_TOKEN -u AZURE_OPENAI_ENDPOINT \
    deep-research research --query '…' --depth low --detach
exit=3
{"error": {"code": "no_provider",
           "message": "The research verb needs the Perplexity backend and no credential is configured.",
           "remedy": "Set PERPLEXITY_API_KEY, or put perplexity in ~/.config/amplifier-research/credentials.toml …"}}
```

**A failed child says so, names why, and points at what was retained.** Real failure,
invalid Perplexity key → `dr-4aee4213` (`raw/09-q4b-failed-status.json`):

```json
"status": "failed", "stage": "gather", "stages_complete": 1,
"liveness": {"state": "final", "why": "the run finished with status 'failed'", "poll_again_in_seconds": null},
"failure": {"code": "backend_error",
            "message": "The Perplexity backend failed: AuthenticationError: Error code: 401 - … 'invalid_api_key' …",
            "remedy": "Check the service status and the credential, then run again. A run that failed keeps whatever it gathered before failing.",
            "stage": "gather"}
```

No `status: running, failure: null` forever. Both fact-check failures reported identically
(`raw/26`, `raw/28`).

**A killed child is reported `abandoned`, not `running`** — `kill -9` on the pid mid-scope
(`raw/25-q4c-status-immediately-after-kill.json`):

```json
"liveness": {"state": "abandoned", "poll_again_in_seconds": null,
             "why": "the run record says 'running' but process 3135639 is gone, so nothing is going to finish it. Whatever reached disk is all there will be."}
```

**A healthy run reports progress honestly.** Poll trace and the run's own event log agree
with the stage record to the second (`raw/30-q4-healthy-events.txt`): scope 7 s → gather 4 s,
9 sources, one `search_web` call at $0.0025 → synthesise 24 s → report → complete, total
34 456 ms. `not_yet_true` was present and accurate on every acceptance.

## D1 — but a failed run under-reports what it spent, in the same breath as admitting it

`dr-dd376b7d` (the licence question) failed `attempts_exhausted` at synthesise.
`status` says (`raw/10-q1-run1-status.json`):

```json
"failure": {"code": "attempts_exhausted",
            "message": "The synthesise stage was rejected 3 times and the attempt budget is spent."},
"usage": {"attempts": 1, "attempts_discarded": 0,
          "cost_usd": "0.02683", "discarded_cost_usd": null,
          "tokens_in": 12515, "tokens_out": 1917}
```

`attempts_discarded: 0` and `"rejected 3 times"` are in **the same document**. The run's own
`attempts.json` has the real numbers:

| attempt | reason | cost | tokens_out |
|---|---|---|---|
| 1 | "no JSON document was found in the reply" | $0.141678 | 8 653 |
| 2 | same | $0.116697 | 6 975 |
| 3 | same | $0.085572 | 4 900 |

**Reported $0.02683. Actually spent $0.370777.** A 13.8x under-report, and every dollar of
the difference is money thrown away — exactly the number a caller needs to decide whether
to retry. Across this whole sweep: tools reported **$0.1896**, real spend **$0.5336**
(`raw/40-cost-ledger-final.txt`).

This directly contradicts the tool's own estimator caveat, printed on every `estimate` call:

> *"A stage that fails validation is retried AND BILLED FOR EVERY ATTEMPT … **The run's own
> run.json records what was actually spent, including attempts_discarded and
> discarded_cost_usd.**"*

Root cause, exact: `research-core/staging.py:56-78` computes `attempts_discarded` and
`discarded_cost_usd` correctly — but only on `StageResult`, which is **only constructed on
success** (`staging.py:149`). The exhausted path raises `AttemptsExhausted` (`staging.py:168`),
and the handler at `tools/deep-research/src/deep_research/research.py:510-523` writes
`attempts.json` and calls `writer.fail(...)` — **and never calls `writer.record_usage(...)`**.
The arithmetic is right and unreachable on the only path where it matters.

`packages/research-core/tests/test_reply_capture.py:165-180` tests `record_usage` with
discarded numbers handed to it directly — the writer's arithmetic, not the failure path that
would feed it.

> ★ **Insight:** `staging.py` computes the discarded cost on `StageResult`, and `StageResult`
> only exists when a stage *succeeds* — so the field designed to report wasted money is
> structurally absent from every run that wasted any. Principle: put the accounting on the
> path that fails, not on the value returned when it doesn't.

## D6 — the repair loop fed back the wrong finding, three times

All three rejected replies contained a JSON document (`raw/11-rejected-json-analysis.txt`).
The parser is not at fault — each is genuinely unparseable, on one specific thing: an
**unescaped `"` inside a string value** (`… The only "package" source given …`), which
survives fence-stripping and `strict=False`:

```
synthesise-01: json.loads FAIL -> Expecting ',' delimiter: line 3 column 2716
synthesise-02: starts '```json' ; after stripping the fence -> STILL FAIL, same cause
synthesise-03: idem
```

The defect is the *feedback*. `research-core/staging.py:126-130` sends back, for any parse
failure:

> "Your reply carried no JSON document. Return ONLY the JSON document asked for, **with no
> prose around it**."

The model's error was never prose-around-it. Attempts 2 and 3 duly acted on that
instruction — they **added a ```json fence** and kept the unescaped quote. The module's own
docstring states the rule this violates: *"a rejection feeds the SPECIFIC finding back into
the next attempt. 'Try again' buys a second attempt at the same mistake."* Three attempts,
one mistake, $0.34, a generic finding.

**This is a tool defect, not a model being wrong on a hard question.** The model produced a
good answer three times (the rejected brief correctly identifies librosa=ISC,
pedalboard=GPLv3, essentia=AGPLv3, matchering=GPLv3 and is *scrupulous* about which of
those the evidence actually supports). The `json.JSONDecodeError` already names the byte
offset and the expected token; passing that through would let the model fix the one
character. The `NoStructureFound` message drops it and substitutes a guess.

## D7 / D8 — two smaller contract deviations

**D7 — error envelopes go to stderr, the docs say stdout.** Both tools' `--help` state:
*"One JSON document on stdout. Success is `{"result": …}`; failure is `{"error": …}` with a
non-zero exit. Progress and diagnostics go to stderr, never stdout, so you can parse one
without filtering the other."* Measured on three error classes
(`raw/07-error-streams.txt`) — every one has **empty stdout** and the envelope on stderr:

```
status dr-nope                     exit=1  STDOUT: (empty)  STDERR: {"error": {"code": "run_not_found", …}}
check-claims (no claims)           exit=2  STDOUT: (empty)  STDERR: {"error": {"code": "usage", …}}
research --backend bogus           exit=2  STDOUT: (empty)  STDERR: {"error": {"code": "usage", …}}
```

An agent following the documented contract (`parse stdout`) sees nothing at all on failure.
The envelopes themselves are excellent — accurate `code`, `message` and actionable `remedy`
every time. Only the stream is wrong. (Either behaviour is defensible; they disagree.)

**D8 — refusals carry no `affordances`.** Every verb document ends with: *"**A refusal
carries `affordances` too** — named next moves, each free and each needing no credential,
so being refused is never a dead end."* None of the five refusals captured in this sweep
(`raw/06`, `raw/07`) carried an `affordances` key. Accepted `--detach` responses do.

## Observation, not a defect: `list` cannot see abandonment

`deep-research list` shows the `kill -9`'d run as `"status": "running"` with no liveness
field (`raw/39-list-fields.txt`) — its documented Result set genuinely has no liveness, and
`--help` tells you to ask `status <id>` and read `liveness.state`. A caller who reads `list`
alone is misled; a caller who follows the docs is not.

---

# What I could not check

- **`--backend agent`** — every run here used the default `perplexity` backend. The agent
  backend is untested by this sweep.
- **Providers other than Anthropic** — `check` reports openai/gemini credentials present but
  their client libraries absent, so only the anthropic path was exercised. (This matches the
  standing note that only one of four backends has ever been called live.)
- **Whether the s1 404 is systematic or one bad URL.** One run, nine sources, one dead. The
  *absence of any liveness check* is proven from source; the base rate of dead citations is
  not — that needs several runs and a sweep of every URL in each.
- **`--strict`, `--no-scope` cost claims, `classify`, `render --format markdown`** — out of
  scope for four questions and a small run budget.
- **Whether `deep-research`'s synthesise stage fails on the licence question reproducibly.**
  It failed once, three attempts, same cause. I did not re-run it; the $0.34 bought the D1
  and D6 findings instead.

# Suggested follow-ups (for whoever files these)

1. **D1 first** — `record_usage` on the `AttemptsExhausted` path in both tools. It is the
   only defect here that silently costs money, and the estimator already promises the field.
2. **D2** — one-key fix (`runs.py:445` → `counts`), plus **replace the hand-written
   `fc-9f8e7d6c` fixture with a recording of a real run**. The fixture is the actual defect;
   the null tally is a symptom, and the same fiction will hide the next one.
3. **D3/D4** — resolve `claims_file` to absolute before the handoff, and move the
   `--from-run` existence check into the detached preflight. Then sweep for siblings: any
   other path-shaped or existence-shaped argument crossing the parent/child boundary.
4. **D5** — a `--verify-links` pass (HEAD each source, record status in `sources.json`) would
   turn "9 sources" into "8 reachable, 1 dead" at near-zero cost, and a dead *cited* source
   should cap `confidence`, not be allowed to justify "high".
5. **D6** — pass `JSONDecodeError.msg`/`.pos` through `NoStructureFound` into the repair
   finding.
6. **D7/D8** — pick one stream contract and make the docs and the code agree; attach
   `affordances` to refusals or stop promising them.
