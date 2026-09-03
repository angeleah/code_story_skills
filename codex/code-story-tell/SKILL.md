---
name: code-story-tell
description: Trace one Elixir code path at runtime with CodeStory, explain the observed call flow, and flag trace-grounded redundancy, fan-out, or suspicious values. Use when someone wants to understand or audit an unfamiliar Elixir function from a live development trace. Do not use for static-only code review or non-Elixir projects.
---

# Code Story Tell

Capture one live CodeStory trace without editing project files, explain only what the trace supports, delete the generated log, and verify that the working tree returns to its original state.

## Preconditions

- Work only in a local development environment. Never trace production.
- The project must use Elixir 1.15+, OTP 27+, and include `{:code_story, "~> 0.2", only: :dev}`. Verify this from `mix.exs`, the lockfile, or installed dependency before running. If CodeStory is missing or unavailable in `dev`, explain how the user can add it and stop; do not edit the project.
- Use a public, fully qualified entry point that can run under `MIX_ENV=dev`. Private functions cannot be invoked directly by `mix run -e`.
- Never edit source, test, configuration, or dependency files. Keep all instrumentation in the single `mix run -e` invocation.

## Establish the invocation

Use an exact runnable expression already supplied by the user. If they named only a function or behavior, ask them to choose one approach:

- Supply a fully qualified development expression with real arguments, such as `MyApp.Orders.process_order(1)`.
- Let Codex inspect production call sites, propose an expression, and wait for confirmation before running it.
- Let Codex inspect a test for the entry shape, translate test-only fixtures into seeded development data, propose an expression, and wait for confirmation before running it.

Treat the expression as executable code. Inspect enough surrounding code to identify apparent material side effects such as writes, messages, payments, or external requests. Disclose those effects and get confirmation before executing them. Do not assume `MIX_ENV=dev` makes a call harmless.

Before executing anything, show this warning:

> ⚠️ CodeStory records real argument and return values. If this flow touches passwords, tokens, API keys, or personal data, those values can be written to a trace file and read into this conversation. Choose `:outline` if the data may be sensitive; it captures names and structure without values.

Ask for the detail level unless the user already chose one:

- `:outline` — function and argument names only; no argument values or returns.
- `:short_story` — names, truncated argument values, and returns. This is the usual choice.
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

Remember that CodeStory traces only the calling process. Work performed in spawned Tasks, GenServers, or other processes will not appear. Its default repeat folding and Ecto repository boundaries make repeated sibling calls legible but intentionally hide lower-level detail.

## Cleanup and verify

Cleanup is mandatory after success or failure:

1. Delete `code_story_trace.log` only if it did not exist before the run and this trace attempt created it.
2. Run `git status --porcelain` again and compare it byte-for-byte with the saved baseline.
3. If the status differs, report every difference. Never restore, discard, or delete the user's other changes.

End every attempted trace with this reminder, adapted for the chosen detail level:

> The generated trace log has been deleted. Trace content was read into this conversation; with `:short_story` or `:novel`, that includes captured values. If sensitive data appeared, treat this conversation accordingly. Use `:outline` next time to keep values out of the trace.
