# Observer long-term memory

## Intent

mc-observer (`observer/observer.py`) is an AI character in Minecraft chat. Today its memory is `state.json`: one
opinion score (-10..10) per player shared by every persona, the last 12 notes per player, and the last 25 log
lines. It forgets past sessions, and switching persona (Grimbold to Deborah) carries Grimbold's opinions over
as if they were Deborah's.

The goal is characters that accumulate experience: each persona forms its own view of each player, remembers
moments, grudges, favors, promises and running jokes, and knows what happened on the server even while another
persona was active. It should feel smarter over weeks of play without noticeably raising token usage. The
owner is on the $20 Claude plan, and the bot's `claude -p` calls share limits with their own Claude Code use.

## Goals

- Local SQLite database holding long-lived knowledge per player and per persona.
- Separate opinions per persona per player, on a finer scale.
- Knowledge beyond opinions: memories of several kinds, facts about players, session history, the bot's own
  actions, and persona-level experience not tied to a player.
- Shared world knowledge: every persona knows what happened while other personas were active, but forms its
  own opinions and memories.
- Token usage per call stays roughly flat (target: within ~25% of the baseline being measured now).

## Non-goals

- No change to when the bot calls the model (`worth_model_call`, chattiness roll) or to model choice: Sonnet
  only, one-shot `claude -p` calls, no Haiku gate, no persistent session.
- No semantic/embedding search. Keyword search (FTS5) plus importance and recency is enough to start.
- No new gameplay abilities.

## Anti-goals (would make this a failure even if it works)

- Token usage that climbs noticeably, especially per call or with session length.
- Weakening the safety model: model output still only fills JSON fields that code validates; no model text is
  ever interpolated into an RCON command.
- Personas sharing opinions or reading each other's private impressions.
- A persona claiming to have forgotten something just because it wasn't in the prompt.

## Constraints

- Plain Python 3 stdlib, no dependencies. `sqlite3` with FTS5 is available (SQLite 3.45.1 on the host).
- The model has no tools (`--tools ""`), so it cannot query the database. Code chooses what goes into each
  prompt; the model writes memories back through JSON fields in its reply.
- `observer.py` is multithreaded: context is gathered on `self.pool` (8 workers), and the main loop handles
  batches. Database access must be safe across those threads.
- Follow `observer/AGENTS.md`: validate every model value in code, no code comments unless the flow is
  complex, restart only the observer (`./start`), never the Minecraft server.
- The token logging already added (`usage in=... cache_write=... cache_read=... out=... prompt_chars=...`)
  is the baseline. Keep it.

## Decided

### Storage

- One file, `observer/memory.db`, gitignored. WAL mode.
- Everything in `state.json` moves into the database: players, opinions, notes, gifts, takes, cooldowns,
  history. After a one-time import on first start, `state.json` is no longer read or written (leave the old
  file on disk; the owner can delete it).
- Events are never deleted.

### Persona identity

A persona is identified by its file slug: the target of the `persona.md` symlink (`personas/deborah.md` ->
`deborah`). Display name comes from the file's `# ` heading as today.

### Schema (columns are the intent; exact types, indexes and names are the builder's)

| Table | Holds |
|---|---|
| `players` | name (primary key), first_seen, last_seen, total_playtime_s |
| `sessions` | player, joined_at, left_at, summary (neutral, shared across personas, written by logoff review) |
| `events` | t, kind (chat, death, advancement, join, leave, observed, bot_say, bot_action), player (nullable), persona active at the time, text, importance. FTS5 index on text |
| `facts` | player, key, value, updated_at, source (model or code). Unique on (player, key); a new value overwrites. Cap ~15 per player |
| `relationships` | persona, player, opinion (-100..100, starts 0), familiarity (interaction count), impression (<=300 chars), nickname (<=24 chars), last_interaction |
| `opinion_changes` | persona, player, t, delta, note (<=80 chars). Private to the persona; replaces the old shared notes |
| `memories` | id, persona, t, kind (moment, grudge, favor, promise, joke), text (<=150 chars), importance (1..10), status (open, resolved), last_recalled_at, recall_count. FTS5 index on text |
| `memory_players` | memory id, player. Links a memory to any number of players; a memory with no links is persona-level experience |
| `actions` | t, persona, player, kind (give, take, summon, potion, title, smite, effect, world), detail (JSON), diamonds (for the hourly diamond budget) |
| `cooldowns` | key, last_used |

### Scales

- Opinion: -100..+100, starts at 0 for every persona. Per-event delta clamped to -15..+15 in code.
- Importance: 1..10. Code clamps. The prompt gives an anchored rubric because models cluster scores: 1 trivia,
  4 notable, 7 a story you'd retell, 10 a defining moment ("betrayed a friend", "killed the dragon").
- Code-assigned defaults for events: chat 1, join/leave 1, observed 2, bot_say 1, death 2, first death in a
  dimension 3, advancement 3 (challenges 4), bot_action by kind (smite, creeper, take 4; give, hostile summon,
  world 3; everything else 2). Builder may tune.

### Migration from state.json

- Old notes become shared history: import each as an `events` row (kind `observed`, player set, importance 3)
  so all personas can find them through the relevant-past-events search. Do not turn them into any persona's
  memories.
- All persona opinions start at 0. Old opinion scores are not carried over.
- Gifts and takes become `actions` rows (persona unknown: use `legacy`), so the diamond budget keeps working.
- Cooldowns and the 25-line history import as-is.

### Writing

Every chat line, game event, observation, bot line and executed action is written to `events` (and `actions`
for actions) by code, no model involved.

The model's reply JSON gains two optional fields alongside `opinion`:

```json
"memory": [{"players": ["Name", "Other"], "kind": "moment|grudge|favor|promise|joke", "text": "...", "importance": 6}],
"fact": [{"player": "Name", "key": "base", "value": "near the spawn river"}]
```

Code validates: every player must match `PLAYER` and be known (empty or missing `players` means persona-level
experience); kind from the allowlist; text and values trimmed and length-capped; importance clamped; at most 2
memories and 2 facts per reply. Memory and fact text are only ever stored and shown back in prompts, never put
into commands.

The existing `opinion` entries keep their `note`; it goes to `opinion_changes` (private to the persona), never
to the shared `events` table.

Open items (promise, grudge, favor with `status = open`) are capped at 5 per persona per player. Saving a sixth
resolves the oldest lowest-importance one. Open items older than 30 days are resolved automatically.

### Reading (prompt assembly)

Items, in budget priority order (highest first, across all involved players):

1. "Since you were last active" for the active persona, only when another persona has been active since this
   persona's last call: neutral session summaries plus shared events with importance >= 3 from that span
   (including other personas' bot actions), newest first. Its own sub-cap: ~600 of the ~2,200 characters.
   This is how Deborah learns what happened on Grimbold's shift, including mid-session.
2. Per involved player: opinion, familiarity, nickname, and the last 2 `opinion_changes` notes.
3. Per involved player: open promises, grudges and favors (at most 5 by the cap above).
4. Per involved player: impression.
5. Top-ranked memories linked to any involved player (3-5 per player; a memory linked to several involved
   players appears once). Rank = importance, decayed by time since created or last recalled, boosted by an FTS5
   keyword match against words in the new batch. Formula is the builder's call; requirement is that a
   high-importance month-old memory beats a trivial one from yesterday. Update `last_recalled_at` and
   `recall_count` for memories included.
6. One or two persona-level memories (no linked players) for the active persona.
7. Per involved player: facts.
8. Per involved player: relevant past events. FTS5 search on `events` for that player, matching words in the
   new batch, importance >= 2, older than the recent log; top 2. This is the "remember when" path for shared
   history, including the notes imported from `state.json`.
9. Per involved player: last session summary.
10. Per involved player: recent actions by any persona toward them.

The recent log shrinks from 25 lines to the last 12 events.

Existing per-player context (vitals, stats, wealth summary, diamond budget, inventory for speakers) stays as is.

### Budget

- Memory context (everything above except the existing per-player block and recent log) is capped at ~2,200
  characters per call (~600 tokens at ~4 chars/token). Calibrate the ratio from the `usage` log
  (`prompt_chars` vs input tokens).
- Fill in the priority order above; each item is included whole or skipped, never truncated mid-text. Nothing
  is exempt from the cap: "always included" means "filled first". If the budget runs out, everything after it
  is dropped for that call.
- Per-item length caps on write keep totals predictable.

### Prompt instructions

`RULES` gains a memory section: what each scale means with the anchored rubric; save meaning, not play-by-play
(the event log already records everything); the memories shown are a selection, so never claim to have
forgotten something; "since you were last active" items are neutral facts to react to through this persona's
own values; the `memory` and `fact` fields and their limits. Opinion instructions change to the -100..100
scale with -15..+15 deltas.

### Logoff review

When a player leaves (`left the game`), one extra Sonnet call, run off the main loop so replies aren't blocked:

- Input: that player's session events, capped at ~6,000 characters. Always include their deaths and
  advancements, everything the bot did to them, and chat from them or addressed to the bot; fill the rest with
  other chat, most recent first; end with a count of omitted lines (e.g. "+212 lines omitted"). Plus the
  active persona's current impression, nickname, open items, top memories, and this session's
  `opinion_changes` for that player.
- Output JSON: neutral session `summary` (<=300 chars, shared), updated `impression` (<=300 chars, this persona
  only), optional `nickname` (<=24 chars, this persona's name for the player), optional `merge` (memory ids to combine into one new memory, e.g. five "died to zombies" into one
  running joke), optional `rescore` (memory id + new importance), optional `resolve` (memory ids whose promise,
  grudge or favor is settled).
- Code validates ids belong to this persona and are linked to this player, caps lengths, clamps scores. A
  merged memory keeps the union of the originals' linked players.
- The persona active at logoff does the review, even if the persona changed mid-session.
- On observer startup, close any open sessions for players not currently online (no review for those).
- Skip the review for sessions shorter than ~2 minutes or with no events beyond join/leave.

## Left to the builder

- Exact SQL types, indexes, and how DB access is made thread-safe (per-thread connections, a lock, or routing
  all writes through one thread).
- Ranking formula details and decay rate.
- Prompt wording, as long as it covers the points above.
- How memory ids are shown to the model in the logoff review (short refs like `m12` are fine).
- Updating `observer/AGENTS.md` (Memory section, Layout table, `state.json` references) and the repo
  `.gitignore` (`observer/memory.db*`).

## Build order

Four checkpoints, one commit each, with a dry run before moving on:

1. Database and migration: schema, one-time import of `state.json`, cooldowns/gifts/takes moved over. The bot
   behaves exactly as before.
2. Write paths: event and action logging, `memory`/`fact`/opinion-note handling, open-item cap.
3. Prompt assembly: budget order and cap, ranking, FTS searches, catch-up after a persona switch, the `RULES`
   memory section, 12-line recent log.
4. Logoff review off the main loop, then `observer/AGENTS.md` and `.gitignore` updates.

## Deployment

- The live bot runs from the main checkout (`/home/myang/Source/mc-server/observer`). `state.json` and the
  `persona.md` symlink exist only there (gitignored). `./start` runs `observer.py` from its own directory, so
  running it from a worktree would start the bot with no state and the default persona.
- Develop and dry-run against a copy of the real `state.json`. Deploy only after the work is merged to `main`,
  by running `./start` in the main checkout.
- Restarting the observer is fine; never restart the Minecraft server.

## Verification

- Dry-run the prompt builder against a populated database and print the assembled prompt and its character
  count; confirm the memory section stays under budget with 1 and with 6 involved players.
- Run the observer for a play session; compare `usage` lines against the baseline collected before the change.
  The ~25% target applies to regular reply calls: uncached input plus output tokens per call, compared to the
  baseline average. Report separately: cached reads (the longer `RULES` lands there), and the logoff reviews
  (count and tokens per day), so the owner can judge total usage.
- Switch persona mid-session and confirm the new persona sees "since you were last active" but not the previous
  persona's opinions or impressions.
- A player leaves: confirm a session summary and updated impression are written, and replies weren't blocked
  while the review ran.
