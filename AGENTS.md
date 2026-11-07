# AGENTS.md

mc-observer: an all-seeing AI character in Minecraft chat (Cap'n Claude, Grimbold, Deborah the HOA president,
...). It watches chat and game events, keeps an opinion of each player, and acts on gameplay (gifts, mobs,
potions, smites, time/weather). Plain Python 3 stdlib, no dependencies; runs on the host as a systemd user unit.

## Commands

```sh
./switch                 # list personas, show the active one
./switch deborah         # change persona; live on the next reply, no restart
./start                  # (re)start the systemd unit `mc-observer` from this directory
systemctl --user stop mc-observer
journalctl --user -u mc-observer -f     # each model call with timing + JSON, plus refusals
python3 -I -m py_compile observer.py    # syntax check
```

After editing `observer.py`: `python3 -I -m py_compile observer.py && ./start`, then watch the journal.
Persona edits need no restart. The unit is transient (`systemd-run`) and does not survive a host reboot; rerun
`./start`. Lines logged during the ~1s restart are missed.

Dry-run a persona without touching the server (prints the model's JSON, executes nothing):
```sh
python3 -I -c "import sys; sys.path.insert(0,'.'); import observer as o
o.PERSONA_FILE = o.HERE/'personas/grimbold.md'; b = o.Observer(); b.refresh_online()
print(b.ask_model(o.persona(), 'Online: X\n\nNEW (react to these, or stay silent):\nX was slain by Zombie'))"
```

## How it works

```
docker logs -f minecraft ----\
vitals poller (every 45s) ----> queue -> worth a model call? -> build prompt -> claude -p --model sonnet
  starving / hurt / dimension /                                                   (one-shot, no tools, ~3-4s)
  deep / quiet spell                                         validate each JSON action -> RCON -> server
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
| `state.json` | Memory: opinions/notes per player, gifts, takes, cooldowns, last 25 log lines (gitignored) |

## Personas

```
# Display Name                   <- chat prefix; name words >4 letters also trigger replies
                                 
color: gold yellow               <- name color, text color (Minecraft color names)
aliases: captain, cap'n, pirate  <- extra whole-word triggers
chattiness: 0.3                  <- odds of reacting to chat not aimed at it (0-1)

Voice. "What you value:" (drives opinions). "Powers:" (generosity vs chaos, favored actions).
```

Current: `pirate`, `grimbold`, `deborah`, `algorithm`, `rummage` (loves poor players), `mortimer`, `gary`.
Describe tone rather than exact catchphrases: the model parrots example lines. Make "What you value" specific;
inverted values are what make characters distinct. No sexual content: the chat is shared with other players
(the user asked once; it was declined).

## Memory

`state.json` `players.<name>`: `opinion` (-10..10, starts 0) and the last 12 `notes`. The model returns
`opinion` deltas, clamped to -3..+3 per event; prompts show involved players' score + last 5 notes.
Opinions are shared across personas (?: per-persona scores not implemented). Deaths/kills come from hidden
scoreboard objectives `obs_deaths`, `obs_kills`, `obs_pvp`, created at startup. Edit `state.json` only while
the unit is stopped; it's rewritten after every event.

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

## Adding an ability

1. Add the field to the JSON schema at the end of `RULES`, plus one line on when to use it.
2. If it has a cooldown, add it to `avail` in `handle()` so the model knows it's usable.
3. Add a handler and register it in `execute()`. Validate every value before `rcon()`: player in `self.online`,
   ids against an allowlist or strict regex, counts clamped, cooldown via `self.ready()`/`self.used()`.
   Never interpolate model text into a command unvalidated. Model on `potion()` or `summon()`.

## Don'ts

- Don't restart or redeploy the Minecraft server as part of observer work (see `../AGENTS.md`).
- Gamerules, difficulty, op/deop, gamemode, kick/ban/whitelist, teleport are not abilities; add only if the user
  explicitly asks, with the same validate-in-code pattern.
- No comments in code unless the flow is complex.
