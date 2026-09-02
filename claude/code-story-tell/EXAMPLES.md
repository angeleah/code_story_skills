# Example — finding an N+1 in a reminder-sending flow

A worked run of `/code-story-tell` against the [Confab demo app](https://github.com/angeleah/code_story_examples), where `Confab.Notifications.send_event_reminders/1` re-fetches the same event once per registrant. The trace shape below is what the code actually produces; the exact counts depend on your seed.

---

**You:**

```
/code-story-tell Confab.Notifications.send_event_reminders(2)
```

**The skill:**

> ⚠️ Heads up — CodeStory records real argument and return values. If this flow touches passwords, tokens, or personal data, those values get written to a trace file and read into this conversation. If the data is sensitive, choose `:outline` below.

Then asks (one prompt, two questions):

- **How should I run it?** → _I'll give you a runnable call_ (you already passed `send_event_reminders(2)` — a seeded livestream event with 500 registrants).
- **How much detail?** → **`:outline`** (the redundancy is a _shape_ problem, and you don't need any user data on screen to see it).

It snapshots the tree, then runs a single throwaway command:

```bash
MIX_ENV=dev mix run -e 'CodeStory.tell(fn -> Confab.Notifications.send_event_reminders(2) end, output: :file, detail: :outline)'
```

Reads back `code_story_trace.log`:

```
--- CodeStory Trace ---
Confab.Notifications.send_event_reminders(event_id)
  Confab.Registration.list_registrations_for_event(event_id)
  Confab.Notifications.deliver_reminder ×500
    Confab.Events.get_event!(id)
    Confab.Notifications.get_mail_settings!(event_id)
    Confab.Accounts.get_user!(id)
    Confab.Notifications.deliver_to(attendee, message)
--- End Trace ---
```

**Findings:**

> `deliver_reminder` folds to **`×500`** — it runs once per registrant. Sitting **inside** that fold are `Events.get_event!(id)` and `Notifications.get_mail_settings!(event_id)` — so each of those fires **500 times**, once for every person, all for the same event. That's a textbook **N+1**: the parent is re-fetched for every child.
>
> `Accounts.get_user!(id)` is also inside the fold (500×), but that's _not_ redundant — it's a different user each time (real work).
>
> **Worth confirming:** hoisting `get_event!` / `get_mail_settings!` up to `send_event_reminders` (fetched once, passed down) would take those 500 fetches each to 1, leaving `get_user!` as the only per-registrant query.

Then it deletes `code_story_trace.log`, confirms `git status --porcelain` matches the pre-run snapshot, and reminds you that the trace values (here, just names — `:outline` captured no values) were read into the conversation.

---

## Notes on this example

- **`:outline` was the right call** — the N+1 is visible from structure alone (`get_event!` nested inside a `×500` fold), and no user data left the code.
- **The completion ceiling:** a 500-registrant run is a lot of traced calls. If the trace comes back empty (it exceeded CodeStory's ~5-second ceiling), re-run once (warm caches) or point at a smaller event — the same N+1 shape shows up as `deliver_reminder ×10` on a 10-registrant event.
- **Findings stay grounded:** the skill reports the observed fold and nesting as fact, and frames the fix as _"worth confirming,"_ not an asserted bug.
