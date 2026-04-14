# MCP Server

trak.studio bietet einen offiziellen [Model Context Protocol](https://modelcontextprotocol.io) Server. Damit kannst du trak.studio-Daten direkt in Claude Desktop, Claude.ai Web, Cursor, VS Code (mit Copilot), und Cowork nutzen — mit einem Klick eingerichtet.

**Server URL:** `https://wxcakuyhddzpmpzjplgg.supabase.co/functions/v1/mcp-server/mcp`

*(Später umzustellen auf `https://mcp.trak.studio/mcp` sobald DNS-CNAME aktiv ist.)*

## 1-Click Install

> **Hinweis:** 1-Click-Deep-Links sind aktuell nur in einigen Clients unterstützt (Claude Desktop ≥ 0.9, Cursor ≥ 0.40). Sonst fallback zur manuellen Config unten.

**Claude Desktop** — [hier klicken](claude://mcp/install?url=https%3A%2F%2Fwxcakuyhddzpmpzjplgg.supabase.co%2Ffunctions%2Fv1%2Fmcp-server%2Fmcp&name=trak.studio)

**Cursor** — [hier klicken](cursor://anysphere.cursor-deeplink/mcp/install?name=trak.studio&url=https%3A%2F%2Fwxcakuyhddzpmpzjplgg.supabase.co%2Ffunctions%2Fv1%2Fmcp-server%2Fmcp)

**VS Code (Copilot)** — [hier klicken](vscode:mcp/install?name=trak.studio&url=https%3A%2F%2Fwxcakuyhddzpmpzjplgg.supabase.co%2Ffunctions%2Fv1%2Fmcp-server%2Fmcp)

## Manuelle Installation

Füge in der MCP-Client-Config ein:

```json
{
  "mcpServers": {
    "trak.studio": {
      "url": "https://wxcakuyhddzpmpzjplgg.supabase.co/functions/v1/mcp-server/mcp",
      "transport": "http"
    }
  }
}
```

Dateipfade:
- **Claude Desktop (macOS):** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Claude Desktop (Windows):** `%APPDATA%\Claude\claude_desktop_config.json`
- **Cursor:** `~/.cursor/mcp.json`
- **VS Code:** in Settings → MCP

## Authentifizierung

Beim ersten Tool-Call öffnet sich der Browser und fragt dich nach Zustimmung. Du meldest dich mit deinem trak.studio-Account an und genehmigst den Zugriff (OAuth 2.1 + PKCE).

**Alternative (Headless / Scripts / Cowork Scheduled Tasks):**
API-Key direkt setzen:

```json
{
  "mcpServers": {
    "trak.studio": {
      "url": "https://wxcakuyhddzpmpzjplgg.supabase.co/functions/v1/mcp-server/mcp",
      "transport": "http",
      "headers": {
        "Authorization": "Bearer trak_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

API-Key generieren: trak.studio einloggen → Einstellungen → KI → API-Key kopieren.

## Verfügbare Tools

**14 Tools in drei Gruppen:**

### Videos

- `list_videos` — Liste aller Videos mit Filter (connection_id, since, content_type, limit)
- `get_video` — Einzelnes Video + letzter Daily-Stats-Snapshot
- `get_performance` — Performance-Zeitreihe (views, retention, ctr, impressions) über N Tage
- `get_retention_curve` — Volle Retention-Kurve (~100 Punkte) mit elapsed time + ratio + percent
- `get_title_history` — Chronologische Titel-/Thumbnail-Änderungshistorie

### Channels

- `list_channels` — Alle verbundenen Kanäle (YouTube/Instagram/Spotify)
- `get_baselines` — Rolling 30-Tage-Baseline (Avg/Median Views, CTR, Watchtime, Outlier-Threshold)
- `top_performers` — Top (oder Bottom mit `order=asc`) Videos nach Metrik

### A/B Tests & Briefings

- `list_ab_tests` — Aktive und abgeschlossene Thumbnail-/Titel-Tests
- `get_ab_test` — Einzelner Test mit per-Variante CTR
- `list_briefings` — Content-Briefings (KI-generierte Titel-Vorschläge + optional Thumbnail-Hypothesen)
- `get_briefing` — Einzelnes Briefing
- `create_briefing` — Neues Briefing anlegen (erfordert `confirm: true`)
- `update_briefing` — Briefing nach Publish updaten (erfordert `confirm: true`)

## Beispiel-Prompts

**Retention-Analyse:**
> "Zeig mir die Top 5 meiner Videos nach Retention bei 30 Sekunden aus den letzten 90 Tagen, und vergleiche das mit den schlechtesten 5."

**Titel-Briefing:**
> "Ich plane eine Folge über Vitamin-D-Mangel. Schau dir meine bisherigen Top-Performer an und generiere 5 Titel-Optionen mit Begründung. Lege das Briefing in trak.studio an."

**Lern-Loop schließen:**
> "Das Briefing mit ID xyz wurde veröffentlicht als Video mit ID abc. Setz chosen_title und linked_video_id."

## Tier-Anforderungen

| Plan | Zugriff |
|---|---|
| Trial | Read-only Teaser: `list_videos`, `get_video`, `list_channels` |
| Pro / Business | Alle Reads + Writes |
| Agency | Zusätzlich Multi-Account-Switching |
| Expired | Nur `list_channels` (Rest liefert Upgrade-Link) |

## Rate-Limits

- 60 Read-Calls pro Minute
- 10 Write-Calls pro Minute

Bei Limit-Überschreitung: Fehlertext mit `retry_after_seconds`.

## Write-Confirmation

Tools die Daten verändern (`create_briefing`, `update_briefing`) verlangen `confirm: true` im Argument. Das verhindert versehentliche Writes durch KI-Halluzinationen. Claude/Cursor wissen mit diesem Convention und fragen dich vor dem Call.

## Zugriffe widerrufen

Unter [app.trak.studio/einstellungen/integrationen](https://app.trak.studio/einstellungen/integrationen) siehst du alle verbundenen MCP-Clients und kannst pro Client den Zugriff widerrufen. Alle Access-Tokens werden sofort ungültig.

## Security

- **OAuth 2.1 + PKCE** als primärer Auth-Weg
- **Access-Tokens** haben 1h Laufzeit mit automatischem Refresh
- **Alle Tool-Calls** werden im Audit-Log protokolliert (Privacy: Args werden SHA-256-gehasht, keine Raw-Titel/Topics persistiert)
- **Writes** erfordern `confirm: true` Flag
- **Dynamic Client Registration** via RFC 7591

## Discovery

Der Server publisht OAuth-Metadaten unter:
```
https://wxcakuyhddzpmpzjplgg.supabase.co/functions/v1/mcp-server/.well-known/oauth-authorization-server
```

## Support

Bei Problemen: [GitHub Issues](https://github.com/ypsum/trak-studio-docs/issues) oder `support@trak.studio`.
