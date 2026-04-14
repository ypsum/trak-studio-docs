# trak.studio — Developer Docs

Öffentliche Dokumentation für Integratoren, die mit [trak.studio](https://app.trak.studio) arbeiten.

## Was ist trak.studio?

YouTube Attribution & Performance-Tracking-Plattform für Content Creator. Zeigt welches Video welchen Umsatz bringt, trackt Retention-Kurven, Traffic-Sources und A/B-Tests — über eine REST-API nutzbar für KI-Tools, Custom-Scripts und externe Integrationen.

## Was findest du hier?

| Ordner | Inhalt |
|---|---|
| [`briefings/`](./briefings/) | High-Level-Briefings für neue Integratoren (z.B. wie man einen KI-Skill an trak.studio anbindet) |
| [`api/`](./api/) | Detaillierte REST-API-Referenz pro Endpoint |
| [`guides/`](./guides/) | Tutorials: Getting Started, Authentication, typische Workflows |

## Quick Start

1. **Account erstellen:** https://app.trak.studio (Self-Service mit 7-Tage Trial)
2. **YouTube-Kanal verbinden:** Einstellungen → Kanäle
3. **API-Key generieren:** Einstellungen → KI
4. **Integration starten:** siehe [briefings/creator-api.md](./briefings/creator-api.md)

## API Base URL

```
https://wxcakuyhddzpmpzjplgg.supabase.co/functions/v1/creator-api
```

Langfristig wird das unter `api.trak.studio` verfügbar sein.

## Authentication

Alle API-Requests erfordern einen API-Key im Header:

```
x-api-key: trak_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Dein Key ist **persönlich und vertraulich** — teile ihn nicht öffentlich und lege ihn in einen Secret-Store (Cowork Secrets, 1Password, env-Variablen).

## Claude-Integration (Claude Desktop / Claude Code / Cowork)

Die REST-API funktioniert aus jedem Claude-Tool via HTTP. Zusätzlich existiert ein lokaler MCP-Server für Claude Code — siehe [`guides/claude-integration.md`](./guides/claude-integration.md) *(coming soon)*.

## Status & Roadmap

- ✅ **Phase 1 (v1)** — Creator-API: Videos, Channels, A/B-Tests, Briefings, Retention-Daten
- 🚧 **Phase 2 (geplant)** — Webhooks, YouTube Test-&-Compare Scraping, Video Description/Tags
- 📋 **Phase 3 (angedacht)** — Remote MCP-Server, Spotify-Integration, Multi-Platform

## Contributing

Feedback, Bugs oder Feature-Requests? Öffne ein [Issue](https://github.com/ypsum/trak-studio-docs/issues) oder schreib direkt an `support@trak.studio`.

## Security

- Keine Credentials im Repo (CI-Check mit [gitleaks](https://github.com/gitleaks/gitleaks))
- Beispiel-Keys sind immer Platzhalter: `trak_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
- Für Security-relevante Findings: `support@trak.studio`

## Lizenz

Dokumentation: CC-BY 4.0 — kannst du frei nutzen, referenzieren, remixen.
Nutzung der trak.studio API: Account + gültiger API-Key erforderlich.
