---
name: code-story-tell
description: Trace one Elixir code path at runtime with CodeStory, explain the observed call flow, and flag trace-grounded redundancy, fan-out, or suspicious values. Use when someone wants to understand or audit an unfamiliar Elixir function from a live development trace. Do not use for static-only code review or non-Elixir projects.
---

# Code Story Tell

Capture one live CodeStory trace without editing project files, explain only what the trace supports, delete the generated log, and verify that the working tree returns to its original state.

## Preconditions

- Work only in a local development environment. Never trace production.
- Check this before any other work in this skill, not by running a trace and seeing it fail: the project must use Elixir 1.15+, OTP 27+, and include `{:code_story, "~> 0.2", only: :dev}`. Verify it from `mix.exs`, the lockfile, or the installed dependency. If CodeStory is missing or unavailable in `dev`, explain how the user can add it and stop; do not edit the project. On a checkout with no dependencies built, warn the user that a full fetch and compile must complete before the first trace runs, since that cost is frequently larger than the question being asked.
- Use a public, fully qualified entry point that can run under `MIX_ENV=dev`. Private functions cannot be invoked directly by `mix run -e`.
- Never edit source, test, configuration, or dependency files. Keep all instrumentation in the single `mix run -e` invocation.

## Decide whether a trace is warranted

A runtime trace earns its setup cost when the shape of execution is the answer. It adds little when the answer already appears in a log line or is reachable by reading a single function. Evaluate the request against the criteria below before establishing an invocation, and state the conclusion plainly. Recommending against a trace is a valid and complete outcome of this skill.

Prefer static reading, and say so, when any of the following hold:

- The error message already names the offending value. A trace will show that value reaching the function that rejected it, which the log has already established.
- The failure is deterministic single-value validation, such as a parse, format check, guard, or changeset error. There is no fan-out to fold and no ordering to observe.

Proceed with a trace when:

- The question concerns how many times or in what order calls occur, including N+1 patterns, redundant fetches, and unexpected fan-out.
- A value is wrong rather than rejected, and reading does not reveal where it was altered.
- The call graph is unfamiliar and a map is needed before making changes.
- The question involves call edges that static analysis cannot see, such as `apply/3`, protocol and behaviour dispatch, or closures passed across module boundaries. A trace observes these edges directly, whereas static analysis must infer them and infers them incorrectly.

## Establish the invocation

Use an exact runnable expression already supplied by the user. If they named only a function or behavior, ask them to choose one approach:

- Supply a fully qualified development expression with real arguments, such as `MyApp.Orders.process_order(1)`.
- Let Codex inspect production call sites, propose an expression, and wait for confirmation before running it.
- Let Codex inspect a test for the entry shape, translate test-only fixtures into seeded development data, propose an expression, and wait for confirmation before running it.

Treat the expression as executable code. Inspect enough surrounding code to identify apparent material side effects such as writes, messages, payments, or external requests. Disclose those effects and get confirmation before executing them. Do not assume `MIX_ENV=dev` makes a call harmless.

Before executing anything, show this warning:

> ⚠️ CodeStory records real argument and return values. If this flow touches passwords, tokens, API keys, or personal data, those values can be written to a trace file and read into this conversation. Choose `:outline` if the data may be sensitive; it captures names and structure without values.

Ask for the detail level unless the user already chose one:

- `:outline` — function and argument names only; no argument values or returns. Use this by default whenever the question concerns how many times, in what order, or what calls what. For those questions it is more accurate, not merely safer: printed values can contain function references naming the function that created them, so counting occurrences in a valued trace counts mentions as calls.
- `:short_story` — names, truncated argument values, and returns. Use when a specific value must be inspected, either because it appears wrong or because the question is whether one value keeps the same parameter name across boundaries. Not a general default.
- `:novel` — complete, untruncated argument values and returns. Restate that all captured values will enter the trace and conversation before using it.

Combine any missing invocation and detail questions into one concise prompt when practical.

## Capture safely

From the target Mix project root:

1. Capture and retain the exact output of `git status --porcelain`. If this is not a Git repository, record that fact.
2. Check whether `code_story_trace.log` already exists. If it does, stop rather than overwrite or later delete a file that this run did not create.
3. Run exactly one trace command, passing the approved expression as one safely quoted shell argument:

   ```bash
   MIX_ENV=dev mix run -e 'CodeStory.tell(fn -> MyApp.Orders.process_order(1) end, output: :file, detail: :short_story)'
   ```

   Replace only the example expression and detail. Run from the project root so the output path is known. Do not add instrumentation to a file. If the expression cannot be represented safely in the shell command, stop and ask for a simpler invocation rather than writing a temporary script.

4. Even if compilation or execution fails, continue to cleanup.

## Classify the result

Check `code_story_trace.log` after the command:

- Present and non-empty: read it completely, then analyze it.
- Missing or empty: state that the trace did not capture. Never describe this state as “no findings,” “no redundancy,” or a clean trace.

For a missing or empty trace, report the most plausible cause supported by the run output:

- The trace exceeded CodeStory's roughly five-second collection ceiling. Suggest a smaller or faster target, smaller seed data, or a positive `depth:` cap.
- The entry made no traced user-defined calls.
- A trace was already active on the calling process, so the expression ran untraced.
- CodeStory traces only modules whose top-level namespace matches the application name, together with its `…Web` sibling. A probe module or anonymous function written for the occasion is therefore not traced, and its absence from the trace says nothing about the code. Build the target from real application functions.
- CodeStory could not start or render the trace; preserve the specific warning or error.

Do not automatically rerun with a different expression, detail level, or depth. That would execute the target again and may repeat side effects; propose the next run and wait for approval.

## Report from evidence

For a successful capture, present the trace before the interpretation. If it is too large to reproduce usefully, say so and provide the smallest faithful excerpts that support every finding.

Look for:

- Repeated calls with identical arguments, especially a fetch nested under a folded `×N` call.
- Repeated calls marked `×N (varies)`, which demonstrate fan-out but not identical inputs.
- Unexpectedly deep or broad call branches.
- An argument or return value that visibly diverges from the expected value the user supplied.

Every finding must cite concrete trace evidence: the function, arguments when visible, repeat count, depth, return, or fan-out. Label interpretations not directly proved by the trace as hypotheses, such as “this looks redundant; confirm whether caching or batching is expected.” Do not call something a bug unless the visible trace itself proves the wrong value or behavior. If nothing stands out in a captured trace, say that honestly.

Remember that CodeStory traces only the calling process. Work performed in spawned Tasks, GenServers, or other processes will not appear. Its default repeat folding and Ecto repository boundaries make repeated sibling calls legible but intentionally hide lower-level detail. Confirm that a repeat count actually renders on a call known to repeat before treating its absence as evidence that no repetition occurred.

When the question concerns scaling rather than a single execution, capture two traces that differ in exactly one input and compare their call profiles. Counts that are identical across both runs represent fixed overhead and are not evidence of a problem, however large they appear. Counts that change with the input are the finding, and their ratio indicates whether growth is linear or worse. Vary only one input between runs. If the larger input also takes a different code path, the comparison conflates scaling with branching and supports no conclusion.

The same comparison serves two further purposes. To map coverage rather than scaling, trace three to five inputs that exercise different branches and union their call profiles. Tracing and static analysis fail in non-overlapping ways, since static analysis misses dynamic edges and tracing misses unsampled branches, so neither constitutes a map on its own. After a refactor intended to preserve behaviour, compare the resulting profile against the pre-change trace; it should be identical except where change was intended.

To determine whether a concept retains its name across boundaries, trace at `:short_story` and search the log for one distinctive value, collecting the parameter names to which it binds. A lopsided distribution is the signal. Renaming at a genuine seam, such as a persisted schema handing off to a query builder, is legitimate translation; renaming within a single context is accidental drift, and the outlier is usually the weaker name.

## Cleanup and verify

Cleanup is mandatory after success or failure:

1. Delete `code_story_trace.log` only if it did not exist before the run and this trace attempt created it.
2. Run `git status --porcelain` again and compare it byte-for-byte with the saved baseline.
3. If the status differs, report every difference. Never restore, discard, or delete the user's other changes.

End every attempted trace with this reminder, adapted for the chosen detail level:

> The generated trace log has been deleted. Trace content was read into this conversation; with `:short_story` or `:novel`, that includes captured values. If sensitive data appeared, treat this conversation accordingly. Use `:outline` next time to keep values out of the trace.
