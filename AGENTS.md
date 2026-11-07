# AGENTS.md

mc-observer: an all-seeing AI character in Minecraft chat (Cap'n Claude, Grimbold, Deborah the HOA president,
...). It watches chat and game events, remembers players per persona (SQLite), and acts on gameplay (gifts, mobs,
potions, smites, time/weather). Plain Python 3 stdlib, no dependencies; runs on the host as a systemd user unit.

## Commands

```sh
./switch                 # list personas, show the active one
./switch deborah         # change persona; live on the next reply, no restart
./start                  # (re)start the systemd unit `mc-observer` from this directory
systemctl --user stop mc-observer
journalctl --user -u mc-observer -f     # each model call with timing + JSON, plus refusals
python3 -I -m py_compile observer.py    # syntax check
python3 -I -c "import sqlite3; d = sqlite3.connect('file:memory.db?mode=ro', uri=True)
[print(tuple(r)) for r in d.execute('SELECT * FROM relationships')]"   # inspect memory (no sqlite3 CLI here)
journalctl --user -u mc-observer | grep usage   # token use per call; call=reply|review
```

After editing `observer.py`: `python3 -I -m py_compile observer.py && ./start`, then watch the journal.
Persona edits need no restart. The unit is transient (`systemd-run`) and does not survive a host reboot; rerun
`./start`. Lines logged during the ~1s restart are missed.

Dry-run a persona without touching the server (prints the model's JSON, executes nothing; uses `memory.db`
in this directory, creating it if missing):
```sh
python3 -I -c "import sys; sys.path.insert(0,'.'); import observer as o
o.PERSONA_FILE = o.HERE/'personas/grimbold.md'; b = o.Observer(); b.refresh_online()
print(b.ask_model(o.persona(), 'Online: X\n\nNEW (react to these, or stay silent):\nX was slain by Zombie'))"
```

## How it works

```
docker logs -f minecraft ----\
vitals poller (every 45s) ----> queue -> log to memory.db -> worth a model call? -> build prompt -> claude -p
  starving / hurt / dimension /        (every event)                      (+ memory)   (sonnet, one-shot, no tools)
  deep / high                                 validate each JSON field -> RCON / memory.db
"left the game" -> logoff review on its own thread (one extra call per session >= 2 min)
```

- Model calls: deaths, advancements, joins/leaves, observations always; chat when it names the persona or an
  alias, when only one player is online, within 2 min of the observer's last line, else `chattiness` odds.
- The model only fills JSON fields; `execute()` maps each to a fixed command template after validating it.
  That split is the safety model: chat (including "ignore your instructions") can change what the model asks
  for, never what the code permits. Keep it for every new ability.
- RCON: `Rcon` class, one socket per thread, auto-reconnect. Container IP from `docker inspect`; port and
  password read at startup from `/data/.rcon-cli.env` inside the container. Never print the password.
- Context is gathered in parallel on `self.pool` (~15ms per player). No AI is involved in gathering.
- `claude -p` uses the user's Claude login. `ANTHROPIC_API_KEY` is deliberately stripped from its env: the
  key in the environment isn't the user's to spend.
- Don't move this back into a Claude Code session with Monitor watchers: notifications arrive late and batched.

## Layout

| Path | What |
|---|---|
| `observer.py` | The bot. Limits/cooldowns at the top, `RULES` prompt, `execute()` and one handler per action |
| `personas/<name>.md` | One character per file, read on every call |
| `persona.md` | Symlink to the active persona (gitignored) |
| `memory.db` | SQLite memory: events, sessions, per-persona relationships and memories, facts, actions, cooldowns (gitignored) |
| `state.json` | Pre-SQLite memory. Imported once into `memory.db` on first start, then never read again (gitignored) |

## Personas

```
# Display Name                   <- chat prefix; name words >4 letters also trigger replies
                                 
color: gold yellow               <- name color, text color (Minecraft color names)
aliases: captain, cap'n, pirate  <- extra whole-word triggers
chattiness: 0.3                  <- odds of reacting to chat not aimed at it (0-1)
powers: unrestricted             <- optional; see "Unrestricted persona" below

Voice. "What you value:" (drives opinions). "Powers:" (generosity vs chaos, favored actions).
```

Current: `pirate`, `grimbold`, `deborah`, `algorithm`, `rummage` (loves poor players), `mortimer`, `gary`,
`pushover` (unrestricted: does whatever anyone asks).
Describe tone rather than exact catchphrases: the model parrots example lines. Make "What you value" specific;
inverted values are what make characters distinct. No sexual content: the chat is shared with other players
(the user asked once; it was declined).

## Memory

`memory.db` (WAL, one connection behind an `RLock` in `Memory`; standard-library `sqlite3` with FTS5). A persona
is its file slug (`personas/deborah.md` -> `deborah`). Spec: `../docs/superpowers/specs/2026-11-07-observer-memory-design.md`.

- Shared by all personas: `events` (every chat line, game event, observation, bot line and action, with an
  importance 1..10 set by code; never deleted), `sessions` (+ neutral summary from the logoff review),
  `players`, `facts` (<=15 per player, a new value per key overwrites), `actions` (diamond budget), `cooldowns`.
- Private to a persona: `relationships` (opinion -100..100, familiarity, impression, nickname),
  `opinion_changes` (delta + note), `memories` (moment/grudge/favor/promise/joke, linked to players through
  `memory_players`; no links = the persona's own experience).
- Writes: code logs events/actions. The reply JSON may carry `opinion` (delta clamped to -15..+15), `memory`
  and `fact` (<=2 each, players must be known, text capped). Open promises/grudges/favors: <=5 per persona per
  player (a sixth resolves the least important), auto-resolved after 30 days.
- Reads: `Memory.context()` builds the MEMORY section, capped at `MEMORY_BUDGET` (2,200 chars), filled in
  priority order and never truncated mid-item: catch-up ("since you were last active", <=600 chars, only when
  another persona logged events since this one's last call), opinion + last 2 notes, open items, impression,
  ranked memories (importance / sqrt(1 + age_days/7), +2 on an FTS keyword match with the batch), 2
  persona-level memories, facts, FTS-matched past events, last session summary, recent actions. Recent log:
  last 12 events.
- Logoff review: on "left the game", `review()` runs on `self.reviewer` (one thread) for sessions >= 2 min
  with real events. Input: the session log capped at 6,000 chars plus the persona's view. Output: shared
  `summary`, private `impression`/`nickname`, `merge`/`rescore`/`resolve` by `m<id>`, validated against that
  persona's memories of that player. Sessions still open at startup are closed without review.
- Deaths/kills come from hidden scoreboard objectives `obs_deaths`, `obs_kills`, `obs_pvp`, created at startup.
- Edit `memory.db` only while the unit is stopped. To reset one persona: delete its rows from
  `relationships`, `opinion_changes` and `memories` (+ orphaned `memory_players`).

## Abilities (enforced in code)

| Field | Command | Limits |
|---|---|---|
| say | `tellraw @a` | one line, <=200 chars, JSON-escaped, >=4s apart |
| give | `give` | online player, `DENIED_ITEM` blocked, <=32, <=8 diamonds-equivalent per player per hour |
| take | `clear` | only from a player who spoke in this batch, <=64 (diamonds <=3), no denied items or diamond gear |
| summon | `summon` | friendly <=5 (1 min); zombie/skeleton/spider/silverfish/slime/witch <=3 (5 min); creeper behind player, ExplosionRadius 1 (15 min) |
| potion | `effect give` | `POTIONS` allowlist with max seconds; levitation <=4s, always paired with slow falling; 1/min per player |
| title | `title <p> title` | <=40 chars, 1/min |
| smite | `lightning <p>` (EssentialsX) | 5 min |
| effect | preset sound/particles | sparkle, sad, thunder, spooky, fanfare |
| world | `time set <x> world`, `minecraft:weather` | day/noon/night/midnight, clear/rain/thunder, 10 min |

## Unrestricted persona

`powers: unrestricted` in a persona header (only `pushover` today) sets `self.free` for that batch. It is read
from the file, never from chat. Same validate-in-code model, wider limits: no cooldowns or diamond budget;
give any item except `FREE_DENIED_ITEM` (command/structure blocks, jigsaw, wither/dragon eggs) up to 64 x6;
take any item or `all` from any online player; summon `FREE_MOBS` (no wither/dragon) up to 10, creepers 3;
potion `FREE_EFFECTS` up to 600s, level 5. Extra fields, refused for every other persona: `gamemode`, `tp`
(player or clamped coords), `xp` (+-1000 levels), `enchant` (`ENCHANTMENTS`, held item), `kill`, `kick`
(reason stripped to `[A-Za-z0-9 .,!?'-]`), `difficulty`. The prompt gets `FREE_RULES` appended. Deliberately
not available even here (user's call): server stop/reload, op/deop, whitelist, ban, gamerules, fill/clone,
`@e`/selectors. Anyone online can command it.

## Adding an ability

1. Add the field to the JSON schema at the end of `RULES`, plus one line on when to use it.
2. If it has a cooldown, add it to `avail` in `handle()` so the model knows it's usable.
3. Add a handler and register it in `execute()`. Validate every value before `rcon()`: player in `self.online`,
   ids against an allowlist or strict regex, counts clamped, cooldown via `self.ready()`/`self.used()`.
   Never interpolate model text into a command unvalidated. Model on `potion()` or `summon()`.
4. After it succeeds, call `self.acted(kind, player, detail, text)` so it lands in `actions` and `events`.

## Don'ts

- Don't restart or redeploy the Minecraft server as part of observer work (see `../AGENTS.md`).
- Gamemode, kick, kill, teleport, xp, enchant and difficulty exist only for unrestricted personas. Gamerules,
  op/deop, ban/whitelist, server control and terrain editing are not abilities for anyone; add only if the user
  explicitly asks, with the same validate-in-code pattern.
- No comments in code unless the flow is complex.
