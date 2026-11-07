---
name: mc-observer
description: Operate mc-observer, the all-seeing AI character on the Minecraft server (Cap'n Claude, Grimbold, Dennis Denies, etc.) that watches chat and game events, remembers what it thinks of each player, and acts on gameplay (gifts, mobs, potions, smites, time/weather). Use whenever the user wants to switch, add, or tweak a personality, change what the observer may do or its limits/cooldowns, inspect or reset its opinions of players, start/stop/restart it, add it to another server, find out why it did or didn't react, or run rcon commands on the minecraft container, even if they just say "the bot", "the god", "the pirate", or a character's name.
---

# mc-observer

The full reference lives with the code. Before doing anything, read:

- `~/Source/mc-observer/AGENTS.md`: how the bot works, instances (one per server), persona format, memory,
  abilities and limits, commands (`switch`, `start`, journal), and how to add an ability safely.
- `~/Source/mc-server/AGENTS.md`: the main server itself, console gotchas, and the rule to check
  `rcon-cli list` and ask before any server restart.

Quick ones (from `~/Source/mc-observer`): `./switch <persona>` (main instance, live, no restart);
`./start [instance]` after editing `observer.py`; `journalctl --user -u mc-observer-main -f` to see what it's doing.
