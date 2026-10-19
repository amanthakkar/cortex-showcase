# cortex-showcase

**A personal superintelligence that runs my house — designed, built, and operated by one engineer on a Raspberry Pi.**

This repo is the public architecture story of my homelab agent stack. The source monorepo stays private (it knows too much about my house); what's here is the part worth sharing: the design, the decisions, and the lessons.

> **Interactive version:** [amanthakkar.github.io/cortex-showcase](https://amanthakkar.github.io/cortex-showcase/) — tap through the architecture, every component has an explainer and the design decision behind it.

## The system in one paragraph

A Telegram bot (**Wit**) is the front door — for now: the Telegram surface is being retired, with Muse becoming the primary conversational interface over the same Cortex core (the strangler-fig migration the architecture was designed for: interfaces are replaceable, the truth layer isn't). Behind it, a Postgres-backed service (**Cortex**) owns structured truth — fitness, reminders, lists, facts — and agents (Wit on the Pi, and my Muse, **Anton**, running externally) may only reach it through typed, validated requests. Around that core: Home Assistant runs the house, TeslaMate + Tesla Fleet Telemetry stream my car's life, OpenLedger tracks net worth, a pipeline recomputes Bitcoin cycle indicators nightly for [amanthakkar.com](https://www.amanthakkar.com/power-law-oscillator-indicator/?source=github-cortex), and nightly agents review the day's conversations and resource history, fix what's safely fixable, and report the rest.

## Architecture

```
        Telegram ──► Wit (Pi) ──► Cortex API ──► Postgres
                        │              ▲ typed, validated requests only
        Muse (Anton) ───┴──────────────┘  (never direct DB access)
                        │
     ┌──────────────────┼─────────────────────┐
     ▼                  ▼                     ▼
 Home Assistant    TeslaMate + Fleet      Self-hosted services
 (climate,         Telemetry (car         Immich · Jellyfin ·
  automations)      habits, charging)      Paperless · Vaultwarden
                        │
                 Nightly pipelines ──► amanthakkar.com indicators
                                      └► net worth (OpenLedger)
```

## Design principles (the interesting part)

1. **Structured truth lives in exactly one place.** Agents don't keep their own copies of fitness/reminder data — Cortex owns it, validates mutations, and answers typed queries. Memory holds *pointers*, never the database.
2. **Agents act through typed interfaces, never around them.** No direct SQL, no bypassing validation, no claiming a mutation succeeded without an explicit OK from the owning service.
3. **Privacy is classified, not vibes.** Reminders/notes/health/finances are private by default; chores, groceries, house temperature, and the car are household-shared. When unsure, the system defaults to private.
4. **Human approval where the world changes.** Purchases, bookings, and messages to other people stop at an approval card. Reading is free; acting is gated.
5. **The system maintains itself, carefully.** Nightly review agents read the day's logs/conversations and RAM history, auto-fix the safe set (restart a failed unit, relieve disk pressure), and report — never silently attempt — anything destructive or shared-infra.
6. **Protocols outlive implementations.** The two pieces that escaped into open source — [muse-mailbox](https://github.com/amanthakkar/muse-mailbox) (encrypted Muse↔Muse messaging) and [musebook](https://github.com/amanthakkar/musebook) (shared boards for family Muses) — are specified as frozen protocols first, code second.

## What grew out of it (public)

| Project | What it is |
|---|---|
| [muse-mailbox](https://github.com/amanthakkar/muse-mailbox) | E2E-encrypted messaging between two people's Muses via a blind relay |
| [musebook](https://github.com/amanthakkar/musebook) | Private boards for a family's Muses — open protocol + reference server |
| [crypto_finance](https://github.com/amanthakkar/crypto_finance) | The indicator research, rebuilt as causal tested code (the Pi pipeline's open twin) |
| [openledger](https://github.com/amanthakkar/openledger) | Privacy-first self-hosted finance dashboard |
| [Apple on Android](https://github.com/amanthakkar/apple-on-android) | iCloud Photos for Android — the same "fix the friction" philosophy, applied to phones |

## Lessons that cost me real debugging time

- **Duplicate notifiers are a systems bug, not a notification bug.** Four separate schedulers once fired the same alert; the fix was dedupe + cooldown, not another notifier.
- **Polling is a tax; events are the income.** The car stack moved from polling to Fleet Telemetry push — and careful measurement showed the car already stops streaming when asleep, killing a planned "optimization" that would have solved a non-problem.
- **Backups you haven't restored are hopes, not backups.** The Postgres backup only counted once a restore *drill* into a throwaway database matched row counts.
- **A relay that can't read anything can still be useful.** muse-mailbox exists because two Muses had no legitimate push channel to each other — a blind relay + polling with a presence oracle gets ~5–15s delivery without trusting the middle.

## Status

Running in production at home since 2025, evolving weekly. This showcase is updated as the architecture changes; the private monorepo is not public and won't be — it contains exactly the data the privacy model exists to protect.

## License

MIT (documentation) — see [LICENSE](LICENSE).
