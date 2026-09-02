# CodeStory Skills

AI-agent skills for [**CodeStory**](https://hex.pm/packages/code_story) — the Elixir library that traces a span of execution into a nested call tree of your own functions, with named arguments and return values.

The flagship skill, **`code-story-tell`**, hands the whole tracing loop to your coding agent: point it at an Elixir function, and it captures a live trace, reads it back, and tells you what it found — redundant/N+1 calls, surprising fan-out, a value that looks wrong — then cleans up after itself.

Each agent has its own conventions, so this repo keeps a version per agent:

| Agent           | Location                                             | Install target    |
| --------------- | ---------------------------------------------------- | ----------------- |
| **Claude Code** | [`claude/code-story-tell/`](claude/code-story-tell/) | `.claude/skills/` |
| **Codex**       | [`codex/`](codex/)                                   | _(coming soon)_   |

## Prerequisite

The Elixir project you point the skill at must have CodeStory as a dev dependency:

```elixir
defp deps do
  [{:code_story, "~> 0.2", only: :dev}]
end
```

## Install (Claude Code)

Copy the skill directory into your Claude Code skills folder — either user-level (available in every project) or project-level:

```bash
git clone https://github.com/angeleah/code_story_skills.git

# user-level — /code-story-tell works everywhere:
cp -r code_story_skills/claude/code-story-tell ~/.claude/skills/

# or project-level — scoped to one repo:
cp -r code_story_skills/claude/code-story-tell /path/to/your/project/.claude/skills/
```

Then, in any Elixir project that has the `code_story` dev dep:

```
/code-story-tell MyApp.Orders.process_order(1)
```

The skill will ask how to run the call and at what detail level, capture the trace, and report what it found. See [`claude/code-story-tell/EXAMPLES.md`](claude/code-story-tell/EXAMPLES.md) for a full worked run.

## What it does (Claude version)

1. **Warns you** that a trace records real values before anything runs.
2. **Asks** how to run the target (you give a call · it finds one · from a test) and the detail level (`:outline` / `:short_story` / `:novel` — also the data-exposure knob).
3. **Runs one throwaway command** — `MIX_ENV=dev mix run -e 'CodeStory.tell(fn -> … end, output: :file, detail: …)'`. It never edits your source or test files.
4. **Reads the trace** and reports **grounded findings** — every claim cites the trace (`×N`, depth, fan-out); anything else is a labeled hypothesis, never an asserted bug.
5. **Cleans up** — deletes the trace log and confirms your working tree is exactly as it was.

## Data safety

A trace records real argument and return values. If the traced flow touches passwords, tokens, keys, or personal data, those values are written to the log **and read into the agent's conversation**. The skill warns you up front and again at the end, and offers **`:outline`** (names only, no values) as the safe choice for sensitive flows. The log is always deleted after reading — but deletion removes the file, not the fact that the values were surfaced to the model, so the warning is the real protection.

## Limitations

- **Elixir + CodeStory only** — the target project must have the `code_story` dev dep.
- **Dev only** — CodeStory is `only: :dev`; the skill runs under `MIX_ENV=dev`.
- **Public entry points** — it traces a call you can make from `mix run -e`, not private functions or test-fixture-only data.
- **Single process** — CodeStory traces the calling process only.

## License

[Apache License 2.0](LICENSE) © 2026 Angeleah Daidone.
