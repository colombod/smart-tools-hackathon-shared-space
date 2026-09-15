# Decisions the spec leaves open

The spec is deliberately silent on several things. Silence means the decision
is yours, and that you own the consequences. These are the ones that came up,
what we chose, and why.

None of this is spec-mandated. Disagree if you have reason to — but decide
deliberately rather than by default.

---

## Which capabilities your model loop may call

**The spec says nothing about this.** There is no field for it; the concept
does not appear.

To be clear about scope: this decides only which tool schemas your
model-backed capability hands to its own loop. It is not a permission system —
every capability stays callable from the library, the CLI, and any agent that
invokes your tool.

What we did: a flag on each catalogue entry, plus a required reason when it is
off.

Left out of the loop, and why:

| Capability | Reason |
|---|---|
| `configure` | The loop does not know the deployment URL. A guess writes a wrong value to disk. A human runs this. |
| `status` | A diagnostic. The loop cannot act on the result; a caller that wants it runs the CLI. |
| The model-backed capability | It is the loop. Offering it inside itself lets it recurse. |

**Do not leave writes out reflexively.** If the bundle's agent could call a
write tool, your model loop should too. Withholding it and then describing the
result as parity was the sharpest correction in the whole conversion.

A test that fails when the reason is empty turns this from an accident into a
decision.

---

## How writes are guarded

**`destructive` does not appear in the spec at all.** That field is ours, and
so is everything built on it.

**What we did in the end: nothing mechanical.** The write reaches shared data
the moment it is called. The only guard is the capability description, which
tells an agent to put the details to the user and get approval first.

We built a `confirmed` argument first and removed it. The model loop passed
the flag automatically, so the caller most likely to write without asking
never met the check; every other caller simply set it; and a flag cannot tell
who set it or whether a human was consulted. The bundle had no such flag
either.

**If your tool has a destructive capability, write the description as though
it is the only thing standing between a model and the write — because it is:**

> WRITES TO SHARED TEAM DATA, attributed to a named person and visible to
> their team. Nothing in this tool can stop the call: if you are an agent, put
> the arguments to the user and get their approval BEFORE calling it.

State plainly that nothing enforces it. A rule described as a mechanism
("raises unless the flag is set") tells an instruction-follower how to make
the error stop; a rule described as a responsibility does not.

**Keep `destructive` meaning one thing.** Ours drifted onto `configure`, which
merges into the caller's own settings file on their own machine — local,
owned, reversible. The CLI duly announced that setting your endpoint "writes
to shared data". Reserve the flag for writes other people can see, or it stops
carrying information.

Real enforcement, if you need it, belongs in Amplifier: a mode's tool policy,
or a human in the loop. A library cannot gate its own caller.

---

## Where configuration lives

**The spec mandates no path, no format, no variable naming.**

What we did: one file, `~/.<tool>/.env`, in `TOOL_KEY=value` form — the same
vocabulary whether a value is exported or stored.

We started with two formats (`.env` plus a YAML file), a path override
variable, and five precedence tiers. All of it went. One format, one file, one
reader.

**Precedence, and this is the non-obvious part:**

```
explicit argument  >  config file  >  environment  >  packaged default
```

The file beats the environment, which is the reverse of the usual convention.
The reason is empirical: during the work we found a variable belonging to the
tool's own family sitting in the environment that nobody in the session had
set — inherited from a parent process. **A config file is a decision someone
recorded; an environment variable is often residue.**

Two consequences:

- **A blank value in the file overrides nothing.** Otherwise copying
  `.env.example`, which is full of empty keys, silently destroys a working
  environment-variable setup. This happened.
- **The variable naming the config directory cannot live in the config file.**
  Pop it from the parsed dict before merging, rather than merely ignoring it,
  so a later refactor that handles all keys uniformly cannot reintroduce it.

**One reader.** Exactly one function in the package touches the environment.
This is the rule most often implemented halfway: converting the obvious
capabilities and leaving the model-provider module reading `os.environ`
directly. A test asserting no other module contains `os.environ` closes it.

Every capability takes an optional config object, so a library caller never
has to set an environment variable.

---

## How authentication is selected

**Spec says nothing.**

What we did: infer from what is present. A key that is non-empty and correctly
prefixed means key auth; anything else means the interactive cloud path.

We first built an explicit mode variable with deprecation machinery and a
warn-once flag. It existed to ease migration from a config format already
deleted, and it was the sole trigger of an unrelated crash. Deleted. We also
built an override to pin the choice; no caller ever passed it, and removing it
took the selection logic to eight lines.

**Document the fallback.** A malformed key silently falls through to the other
path, so someone who pastes a truncated key sees an authentication error
naming a mechanism they were not trying to use. Say so explicitly.

**Never ship a deployment-specific identifier.** A tenant or application id is
configuration, not a constant, however unchanging it looks to you. One was
hardcoded in two places — a default value, and inside an error message's text,
where searching for the obvious occurrence would not find it.

---

## Whether to run setup at install time

**Do not.** The spec's conformance check must pass on a fresh install with
nothing configured, which means you cannot gate first use on configuration.

Also: wheels have no reliable post-install hook, the installing machine is
often not the running machine, and the caller may be an agent for whom an
interactive prompt hangs forever.

The pattern instead:

- **Declare** — `SMART_TOOL.md` names what is needed. The `install` field is a
  reference to documentation, a path or URL, **never a command**. The spec is
  explicit: every field in the manifest is inert.
- **Probe** — `status` reports what is configured and what is missing, runs
  with nothing configured, and never raises. "Not configured" is a successful
  answer.
- **Enforce** — the capability needing the credential fails at call time,
  naming what is missing in the same words the manifest uses.

`status` never raising is easy to break: ours did, because it called a helper
that caught only one narrow exception type. Catch broadly there.

---

## What the CLI returns on a server error

**Spec says nothing.** The decision that matters: keep the server's own
envelope structured rather than flattening it into your message string.

We nest the server's `{code, message, status}` verbatim under `server`,
alongside our own `type`, `message` and `remedy`. A caller that wants to
branch on the server's code does not have to parse JSON back out of a
sentence.

---

## Three that need no argument

**Timeout configurable, default 60s.** The bundle allowed one; hardcoding it
was a divergence. Validate the value and fail loudly rather than falling back.

**No default URL.** A distributable tool must not ship one deployment's
endpoint.

**No compatibility aliases when renaming environment variables.** Supporting
the old name keeps the old vocabulary alive — ours would have reintroduced
another project's name into a tool just separated from it.
