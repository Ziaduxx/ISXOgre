# Developer

Reference material for scripting against OgreBot, ISXOgre, and OgreCraft. These pages cover the public APIs, the positioning and event systems you build encounters on, and the conventions to follow when writing your own scripts.

---

## APIs

- [OgreBotAPI Reference](ogrebot-api.md) — Full API for controlling OgreBot
- [ISXOgre API Reference](isxogre-api.md) — ISXOgre extension types and members
- [OgreCraftAPI Reference](ogrecraft-api.md) — API for OgreCraft

## Systems

- [CampSpot System](camp-spot.md) — Positioning, absolute vs relative campspot (CRCS / ClearRCS)
- [CampSpot Jump System](campspot-jump.md) — Automated jump navigation across gaps and platforms
- [Detrimentals System](detrimentals.md) — Detrimental monitoring and events
- [Ask Query System](ask-query-system.md) — Ask the whole group a question and get one combined answer (all/any true/false, count, JSON)
- [OgreEvents](ogre-events.md) — Event attach/detach pattern for reacting to game actions
- [Preferred Group Members](preferred-group-members.md) — Preferred group & raid member ordering

## Guides

- [Coding Practices](coding-practices.md) — Naming conventions and code style
- [Production Script Patterns](production-patterns.md) — Real patterns, idioms, and known bugs distilled from the live Scripts folder (secondary/informational — API docs remain authoritative)

## Examples

- [Example Scripts](examples/index.md) — Standalone, no-source-needed scripts that talk to the bot through the public `OgreBotAPI`
