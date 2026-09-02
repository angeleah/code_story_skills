---
name: code-story-tell
description: Trace one Elixir code path at runtime with the CodeStory library and report what it does — the call flow, plus any redundant/N+1 calls, surprising fan-out, or a value that looks wrong. Asks how to run the target and at what detail level, captures a live trace via a single throwaway `mix run`, reads it back, reports trace-grounded findings, then deletes the log and leaves the working tree untouched. Use when someone wants to understand or audit an unfamiliar Elixir function at runtime.
disable-model-invocation: true
argument-hint: "[a function or runnable call to trace]"
allowed-tools: AskUserQuestion, Read
---

# code-story-tell — interactive runtime tracer

Capture a [CodeStory](https://hex.pm/packages/code_story) trace of one Elixir code path the user names, read it back, and tell them what it reveals — then clean up completely. Follow the steps **in order**. Never skip the data warning (Step 2), the pre-run snapshot (Step 4), or the cleanup (Step 8) — they run even when the trace fails.

**Prerequisite:** the target project must have `{:code_story, "~> 0.2", only: :dev}` in its deps. If a run fails with `CodeStory` undefined or `:code_story` not available, tell the user to add that dependency and stop — do not try to work around it.

**Never edit source or test files.** The instrumentation lives entirely inside the `mix run -e` string. This is what makes the skill non-destructive; do not deviate from it.

---

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

**Q2 — How much detail?** (this is also the data-exposure knob)

- **`:outline`** — function and argument **names only**, no values. Best for spotting redundancy and shape; **no values leave the code** (names and structure still enter the conversation).
- **`:short_story`** (default) — names, **truncated** values, and returns. Catches most things; moderate exposure.
- **`:novel`** — every value, untruncated. For chasing a wrong value; puts **all** data into the trace and this chat — restate the warning if the user picks this.

## Step 4 — Snapshot the working tree (for non-destructive cleanup)

Before running anything, capture a baseline and remember it:

```bash
git status --porcelain
```

You'll compare against this exact output at the end to prove you left the tree unchanged. (If the project isn't a git repo, note that and rely on "no files edited + log deleted" instead.)

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

  Then still do Step 8.

## Step 7 — Report: the trace, then grounded findings

Show the user the trace, then your reading of it. **Findings contract — do not violate it:**

- Every claim must **cite specific trace evidence** — the repeated call and its `×N`/count, the depth, the fan-out.
- Anything not directly readable from the trace is a **labeled hypothesis** ("this _looks_ redundant — worth confirming"), **never** an asserted bug.
- _"`get_event!` observed inside a `×500` fold"_ is a safe, verifiable claim. _"This is a bug"_ is not — unless the trace shows a wrong value directly.

What to look for: the **same call repeated with identical args** (redundant / N+1), **surprising fan-out**, a **value that looks wrong**. If nothing stands out, say so honestly — a clean trace is a valid result.

## Step 8 — Clean up (non-destructive — required, even on failure)

- Delete the trace log:
  ```bash
  rm -f code_story_trace.log
  ```
- Re-run `git status --porcelain` and confirm it **matches the Step 4 snapshot**. You edited no files, so once the log is gone the two should be identical. If anything else differs, tell the user — **do not** `git checkout`, `git restore`, or discard any of their changes.

## Step 9 — Data reminder (always, at the end)

> Reminder: those trace values were read into this conversation. The log file is deleted, but the values were surfaced to the model — if anything sensitive appeared, treat this conversation accordingly. Next time, `:outline` captures the structure without values.

---

## Notes

- **One process, synchronously.** CodeStory traces the calling process only; work in spawned Tasks/GenServers won't appear. The block form returns the wrapped call's result and writes the file before returning.
- **Boundary calls fold.** Ecto repo calls render as single nodes with named args (`Repo.get!(queryable, id)`); repeated sibling calls fold to `×N`. That folding is what makes an N+1 legible — a fetch nested inside a `×N` node fires N times.
- See **EXAMPLES.md** in this directory for a full worked run (finding an N+1 in a reminder-sending flow).
