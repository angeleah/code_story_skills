# Example — finding an N+1 in a digest-sending flow

A worked run of `/code-story-tell` against an app where `MyApp.Notifications.send_digests/1` re-fetches the same team once per member. Module names here are placeholders — substitute your own. The trace shape below is what such code produces; the exact counts depend on your data.

---

**You:**

```
/code-story-tell MyApp.Notifications.send_digests(2)
```

**The skill:**

> ⚠️ Heads up — CodeStory records real argument and return values. If this flow touches passwords, tokens, or personal data, those values get written to a trace file and read into this conversation. If the data is sensitive, choose `:outline` below.

Then asks (one prompt, two questions):

- **How should I run it?** → _I'll give you a runnable call_ (you already passed `send_digests(2)` — a team with 500 members).
- **How much detail?** → **`:outline`** (the redundancy is a _shape_ problem, and you don't need any user data on screen to see it).

It snapshots the tree, then runs a single throwaway command:

```bash
MIX_ENV=dev mix run -e 'CodeStory.tell(fn -> MyApp.Notifications.send_digests(2) end, output: :file, detail: :outline)'
```

Reads back `code_story_trace.log`:

```
--- CodeStory Trace ---
MyApp.Notifications.send_digests(team_id)
  MyApp.Teams.list_members(team_id)
  MyApp.Notifications.deliver_digest ×500
    MyApp.Teams.get_team!(id)
    MyApp.Notifications.get_mail_settings!(team_id)
    MyApp.Accounts.get_user!(id)
    MyApp.Notifications.deliver_to(user, message)
--- End Trace ---
```

**Findings:**

> `deliver_digest` folds to **`×500`** — it runs once per member. Sitting **inside** that fold are `Teams.get_team!(id)` and `Notifications.get_mail_settings!(team_id)` — so each of those fires **500 times**, once for every person, all for the same team. That's a textbook **N+1**: the parent is re-fetched for every child.
>
> `Accounts.get_user!(id)` is also inside the fold (500×), but that's _not_ redundant — it's a different user each time (real work).
>
> **Worth confirming:** hoisting `get_team!` / `get_mail_settings!` up to `send_digests` (fetched once, passed down) would take those 500 fetches each to 1, leaving `get_user!` as the only per-member query.

Then it deletes `code_story_trace.log`, confirms `git status --porcelain` matches the pre-run snapshot, and reminds you that the trace values (here, just names — `:outline` captured no values) were read into the conversation.

---

## Notes on this example

- **`:outline` was the right call** — the N+1 is visible from structure alone (`get_team!` nested inside a `×500` fold), and no user data left the code.
- **The completion ceiling:** a 500-member run is a lot of traced calls. If the trace comes back empty (it exceeded CodeStory's ~5-second ceiling), re-run once (warm caches) or point at a smaller team — the same N+1 shape shows up as `deliver_digest ×10` on a 10-member team.
- **Findings stay grounded:** the skill reports the observed fold and nesting as fact, and frames the fix as _"worth confirming,"_ not an asserted bug.
