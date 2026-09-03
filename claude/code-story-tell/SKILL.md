---
name: code-story-tell
description: Trace one Elixir code path at runtime with the CodeStory library and report what it does — the call flow, plus any redundant/N+1 calls, surprising fan-out, or a value that looks wrong. Asks how to run the target and at what detail level, captures a live trace via a single throwaway `mix run`, reads it back, reports trace-grounded findings, then deletes the log and leaves the working tree untouched. Use when someone wants to understand or audit an unfamiliar Elixir function at runtime.
disable-model-invocation: true
argument-hint: "[a function or runnable call to trace]"
allowed-tools: AskUserQuestion, Bash, Read
---

# code-story-tell — interactive runtime tracer

Capture a [CodeStory](https://hex.pm/packages/code_story) trace of one Elixir code path the user names, read it back, and tell them what it reveals — then clean up completely. Follow the steps **in order**. Never skip the data warning (Step 2), the pre-run snapshot (Step 4), or the cleanup (Step 8) — they run even when the trace fails.

**Prerequisite — check this first, before Step 0 and before any other work.** The target project must have `{:code_story, "~> 0.2", only: :dev}` in its deps, on Elixir 1.15+ / OTP 27+. Verify that from `mix.exs`, the lockfile, or the installed dependency before you do anything else — not by running a trace and seeing it fail. If it's missing, **tell the user to add it and stop** — don't add it yourself, and don't work around its absence. On a checkout with no dependencies built, warn them that a full fetch and compile has to finish before the first trace runs; that cost is often larger than the question being asked.

**Local development only — never trace production.**

**Never edit source, test, configuration, or dependency files.** The instrumentation lives entirely inside the `mix run -e` string. This is what keeps the skill non-destructive; do not deviate from it.

---

## Step 0 — Talk them out of it, if the question doesn't need a trace

A runtime trace earns its setup cost when the **shape of execution** is the answer. It is close to
worthless when the answer is already sitting in a log line. Check the target against this list before
building a call, and say so plainly if it fails — recommending *against* a trace is a valid outcome of
invoking this skill.

**Don't trace — read the code instead — when:**

- **The error message already names the offending value.** The trace will faithfully show that value
  reaching the function that rejected it. You knew that from the log.
- **The failure is deterministic single-value validation** — a parse, a format check, a guard, a
  changeset error. There's no fan-out to fold and no timing to observe; the failing branch is reachable
  by reading one function.

**Do trace when:**

- The question is **how many times** or **in what order** — N+1s, redundant fetches, surprising fan-out.
  That's what `×N` folding exists for.
- A value is **wrong** rather than rejected, and reading won't show you where it got mangled.
- The call graph is genuinely unfamiliar and you want the map before changing anything.
- You need call edges **grep can't see** — `apply/3`, protocol and behaviour dispatch, closures passed
  across module boundaries. A trace observes those; static analysis infers, and infers wrong here.

## Step 1 — Identify the target

If the user named a target in the invocation (`$ARGUMENTS`) — a function or a call such as `MyApp.Orders.process_order(1)` — use it. Otherwise, ask what code path they want to trace. You need a **public entry point**, not a private function (private functions can't be called from `mix run -e`).

## Step 2 — Data warning (always, before anything runs)

Show this to the user before running:

> ⚠️ **Heads up — CodeStory records real argument and return values.** If this flow touches passwords, tokens, API keys, or personal data, those values get written to a trace file **and read into this conversation**. If the data is sensitive, choose **`:outline`** below — it captures the structure with no values.

## Step 3 — Ask how to run it + how much detail (one `AskUserQuestion`, two questions)

Call `AskUserQuestion` with these two questions together:

**Q1 — How should I run it?** (tracing needs a real, dev-runnable call)

- **"I'll give you a runnable call"** — the user pastes a fully-qualified expression that runs under `MIX_ENV=dev` with real args (a seeded id, not a test fixture), e.g. `MyApp.Orders.process_order(1)`.
- **"Find the call for me"** — search the codebase for a call site of the target to learn its argument shape, then **propose** a dev-runnable expression and get the user's confirmation. Never run an unconfirmed expression.
- **"From a test"** — read a test that exercises the target to identify the entry function, then **propose** a dev-runnable call for it. Test fixtures (e.g. `insert(:event)`) don't exist in `dev`, so map the args to seeded dev data (a real id) and confirm before running.

Whichever they pick, **treat the expression as executable code.** Read enough of the surrounding code to
spot material side effects — writes, enqueued jobs, emails or messages, payments, outbound requests —
then disclose them and get confirmation before running. `MIX_ENV=dev` does not make a call harmless.

**Q2 — How much detail?** (this is also the data-exposure knob)

- **`:outline`** (default for shape and count questions) — function and argument **names only**, no
  values. Correct, not merely safest, whenever the question is *how many times*, *in what order*, or
  *what calls what*. Values are noise there, and actively harmful: printed values can contain function
  references naming the function that created them, so counting occurrences in a valued trace counts
  **mentions as calls**. No values leave the code (names and structure still enter the conversation).
- **`:short_story`** — names, **truncated** values, and returns. Reach for it when you need to *see a
  value* — because one looks wrong, or because you're checking whether the **same** value keeps the
  **same name** across boundaries. Not a general default; moderate exposure.
- **`:novel`** — every value, untruncated. For chasing a wrong value all the way down; puts **all** data
  into the trace and this chat — restate the warning if the user picks this.

## Step 4 — Snapshot the working tree (for non-destructive cleanup)

Before running anything, capture a baseline and remember it:

```bash
git status --porcelain
```

You'll compare against this exact output at the end to prove you left the tree unchanged. (If the project isn't a git repo, note that and rely on "no files edited + log deleted" instead.)

Also check whether `code_story_trace.log` already exists. If it does, **stop** — don't overwrite it, and
don't let Step 8 delete a file this run didn't create. Ask the user to move or remove it first.

## Step 5 — Run the trace (one command, no file editing)

Build `<EXPR>` from Step 3 (fully-qualified, dev-runnable) and `<DETAIL>` from Step 3, then run **exactly one** command:

```bash
MIX_ENV=dev mix run -e 'CodeStory.tell(fn -> <EXPR> end, output: :file, detail: <DETAIL>)'
```

- `MIX_ENV=dev` is **required** — CodeStory is `only: :dev` and is not loaded under `MIX_ENV=test`. `mix run` boots the app (Repo, etc.).
- The trace is written to `code_story_trace.log` in the project root (ANSI-stripped) **only if** it completed with content — see Step 6.
- If the command raises a compile or runtime error, still do Step 8 (cleanup) and then report the failure. Leave nothing behind.

## Step 6 — Classify the result (three states — never conflate them)

After the run, check for `code_story_trace.log` in the project root:

- **Present with content** → read it and continue to Step 7.
- **Present but empty**, or **absent** → the trace **did not capture**. Report that plainly — **never** as "no findings" or "no redundancy." Name the likely cause:
  - The run exceeded CodeStory's ~5-second completion ceiling (a very large trace) → it returns an empty tree and writes no file, raising nothing. Suggest a **smaller/faster target**, a **smaller seed**, or a **`depth:`** cap.
  - The function made no traced user-defined calls.
  - A trace was already active on the process (CodeStory warns and runs untraced).
  - CodeStory only traces modules whose top-level namespace matches the host app (plus its `…Web`
    sibling), so a probe module or anonymous function you write yourself is **not** traced. Build the
    target from real application functions, or the trace will be empty for a reason that has nothing
    to do with the code.
  - CodeStory could not start or render the trace at all — preserve the specific warning or error text
    rather than paraphrasing it.

  Then still do Step 8.

**Don't silently re-run** with a different expression, detail level, or `depth:` cap to get a better
trace. That executes the target again and repeats any side effects. Propose the next run and wait.

## Step 7 — Report: the trace, then grounded findings

Show the user the trace, then your reading of it. **Findings contract — do not violate it:**

- Every claim must **cite specific trace evidence** — the repeated call and its `×N`/count, the depth, the fan-out.
- Anything not directly readable from the trace is a **labeled hypothesis** ("this _looks_ redundant — worth confirming"), **never** an asserted bug.
- _"`get_event!` observed inside a `×500` fold"_ is a safe, verifiable claim. _"This is a bug"_ is not — unless the trace shows a wrong value directly.

What to look for: the **same call repeated with identical args** (redundant / N+1), **surprising fan-out**, a **value that looks wrong**. If nothing stands out, say so honestly — a clean trace is a valid result.

### Two traces beat one — vary one thing and diff

A single trace tells you what happened. It does not tell you what's **inherent** versus what **scales**.
When the question is "does this get worse with more X," capture two traces that differ in exactly one
input, then diff the call profiles rather than reading either one:

```bash
grep -oE "MyApp\.[A-Za-z.]+\.[a-z_]+" traceA.log | sort | uniq -c | sort -rn
```

Anything whose count is **identical** across both is fixed overhead — it is not your problem, however
alarming the raw number looks. Anything that moves with the input is the real finding, and the ratio
tells you whether it's linear or worse.

Keep the two runs honest: change **one** variable. If the larger input also takes a different code path
— a different branch, a different backing store, a different level of scope — the diff conflates
scaling with branching and supports no conclusion.

The same mechanic maps coverage rather than scaling: trace three to five inputs that take **different
branches** and union the profiles. Tracing and static analysis fail in non-overlapping ways — static
analysis misses dynamic edges, tracing misses unsampled branches — so neither is a map alone. And after
a behaviour-preserving refactor, diff the profile against the pre-change trace; it should be identical
except where you meant it to differ.

### Following one datum to find naming drift

To check whether a concept keeps its name across boundaries, trace at `:short_story` and grep the log
for one distinctive value, collecting the parameter names it binds to:

```bash
grep -oE '[a-z_0-9]+: "the-value"' trace.log | sort | uniq -c | sort -rn
```

A lopsided count is the signal. Renaming **at** a genuine seam — a persisted schema handing off to a
query builder — is legitimate translation. Renaming **within** one context is accidental drift, and
the odd one out is nearly always the worse name.

## Step 8 — Clean up (non-destructive — required, even on failure)

- Delete the trace log — **only** if this run created it (Step 4 confirmed it didn't already exist):
  ```bash
  rm -f code_story_trace.log
  ```
- Re-run `git status --porcelain` and confirm it **matches the Step 4 snapshot**. You edited no files, so once the log is gone the two should be identical. If anything differs, tell the user — **do not** `git checkout`, `git restore`, or discard any of their changes.

## Step 9 — Data reminder (always, at the end)

> Reminder: those trace values were read into this conversation. The log file is deleted, but the values were surfaced to the model — if anything sensitive appeared, treat this conversation accordingly. Next time, `:outline` captures the structure without values.

---

## Notes

- **One process, synchronously.** CodeStory traces the calling process only; work in spawned Tasks/GenServers won't appear. The block form returns the wrapped call's result and writes the file before returning.
- **Boundary calls fold.** Ecto repo calls render as single nodes with named args (`Repo.get!(queryable, id)`), and consecutive sibling calls to the same function fold to `×N` — `×N (varies)` when the args differ. Folding is what *should* make an N+1 legible: a fetch nested inside a `×N` node fires N times. Confirm the count actually renders on a call you know repeats before resting a finding on its absence.
- See **EXAMPLES.md** in this directory for a full worked run (finding an N+1 in a reminder-sending flow).
