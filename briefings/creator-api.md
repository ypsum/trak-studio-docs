# trak.studio Creator-API — Integrations-Briefing

**Version:** Phase 1.2 (2026-04-14, nach zweiter Feedback-Runde)
**Adressat:** Cowork-Skills, Claude Code Scripts, andere KI-Tools die auf trak.studio-Daten zugreifen wollen
**Status:** Ready to ship — Implementierung startet auf Freigabe

**Changelog v1.1 → v1.2** (Security + Docs polish):
- `PATCH /briefings/:id` mit fremder `linked_video_id` → jetzt 404 statt 400 (verhindert Enumeration fremder Video-IDs)
- `GET /videos/:id/title-history` ohne Änderungen → liefert `{variants: []}` (explizit dokumentiert, kein 404)
- Performance-Review-Workflow: Title-Changes limitieren Retention-Evaluation, Thumbnail-Changes limitieren CTR-Evaluation (getrennt behandelt)

**Changelog v1.0 → v1.1** (Feature-Runde nach Ruth-Chat-Feedback):
- Retention: zusätzlich `retention_at_90s_pct` (wichtig für längere Intros wie Podcasts)
- Retention-Kurve: `ratio` Field pro Sample für Cross-Video-Pattern-Vergleich dokumentiert
- `top-performers` → `order=asc|desc` Param für Bottom-Performer-Extraktion (Top-3-vs-Flop-3-Lernen)
- Neuer Endpoint: `GET /videos/:id/title-history` (wer hat Titel/Thumbnail wann geändert?)
- Briefings-Tabelle: `proposed_thumbnails` + `chosen_thumbnail_url` optional dazugenommen (Tabellen-Name intern: `content_briefings` statt `title_briefings`)

---

## Was trak.studio liefert

trak.studio ist eine YouTube Attribution & Performance-Tracking-Plattform. Für dein Projekt (z.B. Ruth-Titel-Performance-Lernsystem) stellt trak.studio folgende Datenquellen bereit, alle über eine REST-API erreichbar:

### 1. Video-Stammdaten

Alle Videos eines YouTube-Kanals mit:
- YouTube ID, Titel, Veröffentlichungsdatum, Thumbnail
- Content-Type: `video` / `short`
- Duration (Sekunden)
- Kanal-Zuordnung (connection_id)

### 2. Performance-Zeitreihen (täglich aggregiert)

Pro Video pro Tag:
- **Views, Watchtime, Average View Duration**
- **CTR + Impressions** (via Chrome Extension von trak.studio, nicht über YouTube Public API verfügbar!)
- **Subscribers gained / lost**
- **Traffic-Source-Split:** views_search / views_browse / views_suggested / views_subscriber / views_external / views_other
- **Retention:** `retention_at_30s_pct`, `retention_at_60s_pct`, `retention_at_90s_pct` (wichtig für längere Intros wie Podcasts), plus volle Retention-Kurve als jsonb `[{t: seconds, r: percent, ratio: 0..1}, ...]`. Die Kurve hat ~100 Punkte pro Video. `ratio` ist der elapsedVideoTimeRatio von YouTube (0..1, gleiche Buckets bei allen Videos → gut für cross-video Pattern-Vergleich), `t` ist die absolute Sekunde (pro Video fix).

### 3. Kanal-Baselines (rolling 30d)

Pro Kanal nightly berechnet:
- Average + Median Views pro Video
- Average CTR
- Subs per 100 Views
- Dynamic Outlier Threshold (3x/4x/5x je nach Kanalgröße)

### 4. A/B-Test-Daten

trak.studio rotiert automatisch Thumbnails/Titel für A/B-Tests. Pro Test:
- Test-Typ: `thumbnail` / `title` / `both`
- 3 Varianten pro Test
- CTR + Impressions pro Variante
- Gewinner-Detection

### 5. Content-Briefings (dein Output, speicherbar — Titel + optional Thumbnails)

Du kannst Briefings POSTen und später PATCHen:
- Topic
- Proposed Titles (**freie jsonb-Struktur** — du definierst die Felder: pattern, score, rationale, predicted_retention, ...)
- Proposed Thumbnails (**optional, freie jsonb-Struktur** — z.B. `{url, concept, face_expression, color_hypothesis}`)
- Chosen Title + Chosen Thumbnail URL (nach Publish)
- Linked Video ID (nach Publish)
- Metadata (freies jsonb für alles Kundenspezifische — z.B. `{mode: "medizin", episode_number: 49, published_title_snapshot: "…", end_link_used: true}` für Ruth)

**Wichtig:** trak.studio interpretiert `proposed_titles`, `proposed_thumbnails` und `metadata` NICHT. Das sind dein Lern-Raum. Du kannst beliebige Pattern/Mode/Score-Systeme da reinschreiben und später wieder auslesen.

### 6. Title-/Thumbnail-History (Änderungen nach Publish)

trak.studio erkennt automatisch wenn du einen Titel oder ein Thumbnail auf YouTube geändert hast (via `youtube-sync`). Du kannst die Historie pro Video abrufen:

- `GET /v1/videos/:id/title-history` → Liste aller Änderungen mit Datum, Typ, alter/neuer Wert, Views zum Änderungszeitpunkt

Wichtig für Skills die prüfen wollen: war bei einem Performance-Sample X der ursprünglich gewählte Titel aktiv, oder schon ein geänderter? (Als Backup kannst du auch den ursprünglichen Titel im Briefing-`metadata.published_title_snapshot` selbst sichern.)

---

## Zugriff

### Auth

Jeder Kunde hat einen eigenen trak.studio-Account mit einem API-Key (Format: `trak_xxxxxxxxxxxxxxxx`, 32 Zeichen). Der Key wird als HTTP-Header mitgeschickt:

```
x-api-key: trak_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Jeder API-Key ist automatisch scoped auf den jeweiligen Kunden — du kannst mit Ruths Key keine Daten von Sven sehen (und umgekehrt).

### Base URL

```
https://wxcakuyhddzpmpzjplgg.supabase.co/functions/v1/creator-api
```

Alle Endpunkte sind zusätzlich versioned: `/v1/...`

### CORS

`*` — funktioniert aus Browser, Server, CLI, Cowork Scheduled Tasks.

### Rate Limits

Supabase Edge Functions: ~60 req/min pro Function-Instance. Für die üblichen Use Cases (nightly Reviews, on-demand Briefings) unproblematisch.

---

## Endpunkte

### GET /v1/videos

Liste aller Videos des Kunden.

**Query Parameter:**
- `connection_id` (optional): nur Videos eines bestimmten Kanals
- `since` (optional): ISO-Datum, z.B. `2026-01-01`
- `content_type` (optional): `video` oder `short`
- `limit` (optional, default 50, max 500)

**Response:**
```json
{
  "videos": [
    {
      "id": "uuid-of-trak-video",
      "platform_id": "dQw4w9WgXcQ",
      "title": "Blutwerte: Welche sollte ich messen?",
      "url": "https://youtube.com/watch?v=...",
      "thumbnail_url": "https://i.ytimg.com/vi/.../maxresdefault.jpg",
      "published_at": "2026-03-15T08:00:00Z",
      "content_type": "video",
      "duration_seconds": 1245,
      "connection_id": "uuid-of-connection"
    },
    ...
  ]
}
```

### GET /v1/videos/:id

Einzelnes Video + letzter Performance-Snapshot.

**Response:**
```json
{
  "id": "...",
  "platform_id": "dQw4w9WgXcQ",
  "title": "...",
  "published_at": "...",
  "latest_snapshot": {
    "date": "2026-04-13",
    "views": 12450,
    "watchtime_minutes": 87230,
    "avg_view_duration": 612,
    "ctr": 8.2,
    "impressions": 152000,
    "subscribers_gained": 47,
    "retention_at_30s_pct": 86.0,
    "retention_at_60s_pct": 78.5,
    "retention_at_90s_pct": 72.1,
    "source": "nightly"
  }
}
```

### GET /v1/videos/:id/performance?days=30

Zeitreihe der letzten N Tage.

**Response:**
```json
{
  "video_id": "...",
  "days": 30,
  "snapshots": [
    {
      "date": "2026-03-16",
      "views": 3200,
      "watchtime_minutes": 22400,
      "avg_view_duration": 598,
      "ctr": 7.9,
      "impressions": 40500,
      "subscribers_gained": 12,
      "subscribers_lost": 1,
      "retention_at_30s_pct": 85.2,
      "retention_at_60s_pct": 77.8,
      "retention_at_90s_pct": 71.5,
      "views_search": 480,
      "views_browse": 1120,
      "views_suggested": 890,
      "views_subscriber": 560,
      "views_external": 120,
      "views_other": 30,
      "source": "nightly"
    },
    ...
  ]
}
```

### GET /v1/videos/:id/retention-curve?date=2026-04-13

Volle Retention-Kurve an einem bestimmten Tag.

**Response:**
```json
{
  "video_id": "...",
  "date": "2026-04-13",
  "retention_at_30s_pct": 86.0,
  "retention_at_60s_pct": 78.5,
  "retention_at_90s_pct": 72.1,
  "curve": [
    { "t": 0,  "r": 100.0, "ratio": 0.00 },
    { "t": 12, "r": 95.3,  "ratio": 0.01 },
    { "t": 25, "r": 88.1,  "ratio": 0.02 },
    { "t": 38, "r": 82.5,  "ratio": 0.03 },
    ...
  ]
}
```

`t` ist die absolute Sekunde im Video (pro Video fix), `ratio` ist der `elapsedVideoTimeRatio` von YouTube (0..1, ~100 Buckets bei allen Videos gleich — ideal für Cross-Video-Pattern-Analyse). `r` ist die Retention in Prozent (0-100).

### GET /v1/videos/:id/title-history

Alle Titel- und Thumbnail-Änderungen eines Videos, chronologisch.

**Response:**
```json
{
  "video_id": "...",
  "variants": [
    {
      "detected_at": "2026-03-20T14:12:00Z",
      "variant_type": "title",
      "title_before": "Alter Titel",
      "title_after": "Neuer Titel",
      "thumbnail_before": null,
      "thumbnail_after": null,
      "views_at_change": 8500
    },
    ...
  ]
}
```

Wichtig wenn du prüfen willst: war bei Performance-Snapshot X der ursprünglich gewählte Titel noch aktiv, oder schon ein geänderter? **Wenn keine Änderungen existieren:** `{"video_id": "...", "variants": []}` (leeres Array, nicht 404).

### GET /v1/channels

Liste aller verbundenen Kanäle des Kunden.

### GET /v1/channels/:id/baselines

Rolling 30d Baseline für einen Kanal.

**Response:**
```json
{
  "connection_id": "...",
  "computed_at": "2026-04-14T04:15:00Z",
  "avg_views_per_video": 8500,
  "median_views_per_video": 4200,
  "avg_ctr": 6.2,
  "subs_per_100_views": 0.4,
  "outlier_threshold": 4.0
}
```

### GET /v1/channels/:id/top-performers?metric=retention_at_30s_pct&days=90&limit=10&order=desc

Top- oder Bottom-Videos nach Metrik.

**Erlaubte Metriken:** `views`, `retention_at_30s_pct`, `retention_at_60s_pct`, `retention_at_90s_pct`, `ctr`, `avg_view_duration`, `subscribers_gained`, `watchtime_minutes`

**Order:**
- `desc` (default) → Top-Performers (beste zuerst)
- `asc` → Bottom-Performers (schlechteste zuerst) — ideal für Flop-Pattern-Extraktion / Top3-vs-Flop3-Vergleiche

**Response:**
```json
{
  "metric": "retention_at_30s_pct",
  "days": 90,
  "order": "desc",
  "top": [
    {
      "video_id": "...",
      "title": "...",
      "published_at": "...",
      "thumbnail_url": "...",
      "metric_value": 89.4
    },
    ...
  ]
}
```

### GET /v1/ab-tests?status=active

Liste aller A/B-Tests. Status: `active` / `completed` / `paused`.

### GET /v1/ab-tests/:id

Einzelner Test mit allen Varianten + CTR-Vergleich.

### POST /v1/briefings

Neues Content-Briefing anlegen (Titel und optional Thumbnails).

**Body:**
```json
{
  "topic": "Blutwerte-Folge",
  "proposed_titles": [
    {
      "title": "Welche Blutwerte solltest du messen?",
      "pattern": "konkrete_frage",
      "score": 87,
      "rationale": "Frage-Format performt bei Ruth bisher 22% über Baseline",
      "predicted_retention_30s": 82.0
    },
    {
      "title": "Das sagen deine Blutwerte wirklich aus",
      "pattern": "pain_promise",
      "score": 79,
      "rationale": "...",
      "predicted_retention_30s": 78.0
    }
  ],
  "proposed_thumbnails": [
    {
      "url": "https://example.com/thumb-a.jpg",
      "concept": "face closeup with shocked expression",
      "color_hypothesis": "red text on dark",
      "predicted_ctr": 8.5
    }
  ],
  "metadata": {
    "mode": "medizin",
    "episode_number": 49,
    "topic_tags": ["blutwerte", "labordiagnostik"],
    "published_title_snapshot": "Welche Blutwerte solltest du messen?",
    "end_link_used": true
  }
}
```

`proposed_thumbnails` ist optional — wenn der Skill auch Thumbnail-Hypothesen testen soll. `metadata` ist freies jsonb (für `mode`, `pattern`, `published_title_snapshot` als Backup bei späteren Title-Changes, `end_link_used` für Pattern wie "End-Link-Pflicht" etc.).

**Response (201):**
```json
{
  "id": "uuid",
  "user_id": "...",
  "created_at": "2026-04-14T10:30:00Z",
  "updated_at": "2026-04-14T10:30:00Z",
  "topic": "Blutwerte-Folge",
  "proposed_titles": [ ... ],
  "proposed_thumbnails": [ ... ],
  "chosen_title": null,
  "chosen_thumbnail_url": null,
  "linked_video_id": null,
  "metadata": { ... }
}
```

### GET /v1/briefings

Liste aller Briefings.

**Query:**
- `since` (ISO-Datum)
- `limit` (default 50, max 500)
- `has_chosen`: `true` / `false` — filtert nach linked vs. unlinked Briefings

### GET /v1/briefings/:id

Einzelnes Briefing.

### PATCH /v1/briefings/:id

Briefing nachträglich updaten (typischerweise nach Publish).

**Body (alle Felder optional):**
```json
{
  "chosen_title": "Welche Blutwerte solltest du messen?",
  "chosen_thumbnail_url": "https://i.ytimg.com/vi/.../maxresdefault.jpg",
  "linked_video_id": "uuid-of-trak-video",
  "metadata": { ... }
}
```

Wenn `linked_video_id` gesetzt wird: trak.studio prüft dass das Video dem Kunden gehört (404 sonst — gleicher Status wie "Briefing nicht gefunden", damit externe Clients keine fremden Video-IDs enumerieren können).

---

## Typische Workflows

### Titel-Briefing generieren (on-demand)

```
1. GET /v1/channels → connection_id finden
2. GET /v1/channels/:id/top-performers?metric=retention_at_30s_pct&days=90&limit=10
   → kennt Best-Pattern
3. KI generiert 8-10 Titel, scored intern
4. POST /v1/briefings mit proposed_titles + metadata
5. Canay wählt finalen Titel
6. Nach Publish: PATCH /v1/briefings/:id mit chosen_title + linked_video_id
```

### Performance-Review (Scheduled Task, alle 2 Wochen)

```
1. GET /v1/briefings?since=<2w ago>&has_chosen=true
   → alle Briefings mit verlinktem Video
2. Für jedes Briefing:
   GET /v1/videos/:linked_video_id/performance?days=14
   → echte Retention vs. predicted_retention aus proposed_titles
3. Optional: GET /v1/videos/:linked_video_id/title-history
   → Title-Change: Retention-Evaluation auf Zeitraum vor dem Title-Change
     limitieren (Titel beeinflusst Retention-Erwartung nicht direkt, aber
     auch nicht null — sichere Seite)
   → Thumbnail-Change: CTR/Impressions-Evaluation auf Zeitraum vor dem
     Thumbnail-Change limitieren. Retention ist davon unabhängig.
4. GET /v1/channels/:id/top-performers?metric=...&days=90&order=desc
   → aktuelle Best-Pattern (Top 5)
5. GET /v1/channels/:id/top-performers?metric=...&days=90&order=asc
   → Flop-Pattern (Bottom 5) für Kontrast-Lernen
6. Pattern-Analyse intern (KI-Skill)
7. Update eigene Pattern-Memory-Datei
8. Optional: report an Canay via Mail/Slack
```

### Kanal-Onboarding (manueller Schritt)

```
1. Ypsum legt neuen trak.studio Account an ODER Kunde registriert selbst
2. Kunde verbindet YouTube-Kanal via app.trak.studio/einstellungen/kanaele
3. trak.studio synct automatisch nightly (3 UTC)
4. Ypsum überträgt Kunde-API-Key in Cowork-Secret-Store
5. Cowork-Skill mit Kunde-Key starten
```

---

## Fehlerbehandlung

| Status | Bedeutung |
|--------|-----------|
| 200 | OK |
| 201 | Created (POST /briefings) |
| 400 | Invalid body / query parameter (z.B. unbekannte Metric) |
| 401 | Fehlender/invalider API-Key oder Account inaktiv |
| 404 | Resource nicht gefunden ODER gehört nicht zum Kunden (bewusst same status → kein User-Enumeration) |
| 500 | Interner Fehler (Details nur in trak.studio Logs, nicht in Response) |

Alle Error-Responses sind JSON: `{"error": "Deutsche Fehlermeldung"}`

---

## Was NICHT verfügbar ist (noch)

Damit du weißt wo die Grenzen sind:

- **Real-Time Data / Webhooks:** Aktuell nur Polling. Sync läuft nightly 3 UTC + alle 2h für Videos <72h alt.
- **YouTube Test-&-Compare-Ergebnisse (natives YT-Feature 3-Titel-Tests):** Liefert Google nicht via API. Auf der Roadmap für Browser-Extension-Scraping.
- **Kommentar-Texte/Analyse:** Aktuell nur Kommentar-Count, kein Text-Zugriff.
- **Video-Description/Tags:** Noch nicht über creator-api — kommt in Phase 2.
- **Monthly Listener / Spotify:** Spotify API hat diese Daten abgeschnitten (geparkt).
- **Remote MCP Server:** Aktuell nur REST. MCP Server existiert lokal (stdio) für Claude Code. Remote MCP ist separates Projekt.
- **POST /ab-tests:** Nur Read-Access auf bestehende Tests. Create/Update kommt in Phase 2.

---

## Roadmap (Phase 2+, ohne Timeline)

- Browser Extension erweitern: Scraping von Test-&-Compare, Audience-Profile, Titel/Description-Änderungs-Historie
- Remote MCP Server (Cowork-kompatibel)
- Webhooks auf neue Snapshots
- Video-Description/Tags in API
- Spotify Integration (blockiert durch API-Restriction — Extended Quota Mode beantragen)

---

## Kontakt / Fragen

- Technischer Ansprechpartner: Canay (mail@ypsummedia.de / mail@trak.studio)
- Spec-Dokument (detailliert): `docs/superpowers/specs/2026-04-14-creator-api-design.md` im trak-studio Repo
- Implementation-Plan: `docs/superpowers/plans/2026-04-14-creator-api.md`

**Feedback erwünscht:** Wenn ein Endpunkt fehlt, ein Response-Feld unklar ist, oder ein Use Case nicht abgedeckt — sag Bescheid, wir ergänzen das in Phase 2.
