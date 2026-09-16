# Long-running smart tools: state the obligation, never the mechanism

**Status:** proposal · **Audience:** the spec · **Evidence:** three harnesses, two real
detached runs, ~$4 of live spend · **Author:** colombod

ROADMAP question 1 asks what the default should be for smart capabilities that are long
running or expensive. We now have something better than an opinion: two agent harnesses we did
not design for drove a real detached run of our tool to completion, and a third planned one.
This is what they taught us.

---

## 1. The finding that surprised us most

**Two hosts hit the same wall and invented two different, equally correct ways over it.**

Both faced a hard 120-second per-command limit against runs that took **784** and **658**
seconds.

```
Claude Code    13 commands    9 bounded poll LOOPS   (9 x 10s, break early on final)
Codex          45 commands   31 individual status POLLS
```

Neither was told how to wait, because our documentation said *"wait `poll_again_in_seconds` and
ask again"* and never considered that a host's own call limit might be **shorter than the
interval it was handed**.

**The gap is real — 2 of 2 hosts improvised. The solution is host-specific — neither improvised
the same way.**

### Proposal 1a: the spec states the obligation and refuses to prescribe the mechanism

> A caller must be able to learn whether work is still happening **without blocking past its own
> limit.** How it shapes that wait is its own business.

Had we written a recommended loop shape after watching the first host, we would have blessed
one correct answer and implicitly condemned another. Two samples were the minimum needed to see
that, and we nearly shipped the wrong thing off one.

### Proposal 1b: say that polling is free, because a host cannot infer it

The sentence neither host could have known from our text:

> Polling **more often** than the hint is free and safe. `status` is deterministic, costs
> `$0.00` and needs no credential, so many short checks are exactly as correct as few long ones.

Without it, Codex's 31 polls look wasteful. With it, they are obviously fine. **A tool that
offers free navigation must say the navigation is free**, or callers will ration it out of
misplaced politeness.

---

## 2. `poll_again_in_seconds` is a hint, and we wrote it as an instruction

Our original text read as *sleep this long*. It should read as *new work will probably exist by
then*. Those differ exactly when a host cannot sleep that long inside one call — which is the
only situation where the field matters at all.

### Proposal 2: any "check back later" field is documented as a hint about **when work will
exist**, never as an instruction about **how long to sleep**

---

## 3. Liveness must be separable from progress, and must have a terminal dead state

Both hosts correctly polled `liveness.state` rather than stage names, because we told them to.
The three states earn their keep:

| state | meaning | why it matters |
|---|---|---|
| `growing` | still working | distinguishes *slow* from *dead* |
| `final` | done | the only state where an answer exists |
| `abandoned` | process gone, nothing more coming | **terminal — stop polling** |

Claude Code named the reason precisely: *"that's how I distinguish 'dead' from 'slow,' not a
timeout I invent on my own."* Without a dead state, every host invents its own timeout, and
every one of those timeouts is wrong in a different way.

### Proposal 3: a detached contract carries a liveness field distinct from stage progress, with an explicit terminal dead state

And the half that makes it safe: **whatever reached disk before a process died stays readable.**
A dead run should be salvageable, not a total loss.

---

## 4. The cost story is worse than "declare cost up front"

We have argued a tool should declare cost before a caller commits. We still believe it. But our
own estimator was **2x low** on a cheap run and **6.3x low** on an expensive one, and the
dominant cause was not evidence volume:

```
synthesise  attempt 1   REJECTED   $0.384519
            attempt 2   REJECTED
            attempt 3   accepted
            ~63% of that run bought nothing
```

The retry machinery worked exactly as designed, the caller got a correct answer, **and the
result envelope reported one number with no hint that most of it was discarded.**

### Proposal 4a: declare cost **and the shape of its uncertainty**

A point estimate implies a precision nobody can deliver, because **an estimate cannot know how
many attempts a stage will need.** A range, or an explicit "this may retry and you will be
billed for the attempts", is a promise a tool can actually keep.

### Proposal 4b: a result that retried says so

*"This cost $1.01"* and *"this cost $1.01, of which $0.66 was discarded and retried"* are
different facts. Only the second lets a caller act — lower the depth, change the question, or
report that a stage is unreliable.

---

## 5. The one we did not expect: the host's scepticism is part of the cost

Codex made **nine web-tool calls of its own** to verify and supplement our tool's output, then
presented an answer interleaving our report with sources it had checked itself, marking which
were which.

**It did not treat the smart tool's result as an answer. It treated it as evidence to
corroborate.**

We do not yet know whether that generalises. If it does, every cost model in this ecosystem —
including the one we are proposing above — prices the wrong thing: **the caller's true bill
includes re-verifying work it just paid for.**

### Proposal 5: treat this as an open question the spec should name, not answer

We have one observation from one host on one task. But it points at something none of our
measurements capture, and a tool that makes its evidence *easy to check* may be cheaper in
practice than one that merely asserts a conclusion — even if the second looks cheaper on the
invoice.

---

## What we would ship into the spec tomorrow

1. **The obligation, not the mechanism** — a caller can learn whether work continues without
   blocking past its own limit.
2. **"Check back later" fields are hints**, about when work will exist.
3. **Free navigation is documented as free**, or it gets rationed.
4. **Liveness distinct from progress, with a terminal dead state**, and partial artifacts
   readable after death.
5. **Cost declared with its uncertainty**, and retries visible in the result.

Items 1–4 cost a paragraph each and would have saved two capable hosts from inventing
workarounds. Item 5 is the one we got wrong ourselves, twice, and only caught by arithmetic.

## What this is not

One team, two tools, three harnesses, one language. Every number here is real and every one is
`N` small. The recommendations we are most confident in are the ones where **two independent
hosts behaved the same way without coordination** — and the one we are least confident in is
§5, which rests on a single observation we found interesting enough to write down anyway.
