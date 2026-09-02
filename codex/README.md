# code-story-tell — Codex version

_Placeholder._ A Codex-native version of the `code-story-tell` skill will live here.

It should follow the same contract as the [Claude version](../claude/code-story-tell/SKILL.md):

- **Warn** that a trace records real values before running.
- **Ask** how to run the target (user gives a call · find one · from a test) and the detail level (`:outline` / `:short_story` / `:novel`).
- **Run one throwaway command** — `MIX_ENV=dev mix run -e 'CodeStory.tell(fn -> … end, output: :file, detail: …)'`. **Never edit source or test files.**
- **Three-state log handling** — content / empty / absent; report empty-or-absent as "trace didn't capture," never "no findings."
- **Grounded findings** — cite trace evidence (`×N`, depth, fan-out); hypotheses labeled, never asserted bugs.
- **Non-destructive cleanup** — delete `code_story_trace.log`, confirm the working tree matches the pre-run snapshot.
- **Data reminder** at the end.

Adapt the packaging and invocation to Codex's own conventions.
