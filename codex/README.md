# code-story-tell — Codex version

A Codex-native version of the `code-story-tell` skill lives in [`code-story-tell/`](code-story-tell/).

## Install

```bash
git clone https://github.com/angeleah/code_story_skills.git
cp -r code_story_skills/codex/code-story-tell "${CODEX_HOME:-$HOME/.codex}/skills/"
```

Codex loads skills from `$CODEX_HOME/skills` (defaults to `~/.codex/skills`). If the skill doesn't appear in `/skills`, restart Codex.

## Use

`agents/openai.yaml` sets `allow_implicit_invocation: false`, so Codex won't auto-trigger the skill — name it explicitly:

```
$code-story-tell MyApp.Orders.process_order(1)
```

## Layout

```
code-story-tell/
├── SKILL.md              instructions + name/description frontmatter
└── agents/openai.yaml    display metadata and invocation policy
```

## Contract

It follows the same contract as the [Claude version](../claude/code-story-tell/SKILL.md):

- **Warn** that a trace records real values before running.
- **Ask** how to run the target (user gives a call · find one · from a test) and the detail level (`:outline` / `:short_story` / `:novel`).
- **Run one throwaway command** — `MIX_ENV=dev mix run -e 'CodeStory.tell(fn -> … end, output: :file, detail: …)'`. **Never edit source or test files.**
- **Three-state log handling** — content / empty / absent; report empty-or-absent as "trace didn't capture," never "no findings."
- **Grounded findings** — cite trace evidence (`×N`, depth, fan-out); hypotheses labeled, never asserted bugs.
- **Non-destructive cleanup** — delete `code_story_trace.log`, confirm the working tree matches the pre-run snapshot.
- **Data reminder** at the end.

See [`claude/code-story-tell/EXAMPLES.md`](../claude/code-story-tell/EXAMPLES.md) for a worked run; the trace output and findings are agent-independent.
