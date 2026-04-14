# BriefOps API — Übersicht

Die BriefOps Public API ermöglicht programmatischen Zugriff auf Meeting-Verwaltung, Task-Tracking, KI-Zusammenfassungen und Audio-Transkription.

**Base URL:** `https://your-instance.briefops.io/api/v1`
**API Version:** `1.0.0`
**OpenAPI Spec:** `GET /api/v1/openapi.json`

---

## Quickstart

### 1. API-Key erstellen

API-Keys werden im BriefOps-Dashboard unter **Settings → API Keys** erstellt. Alle Keys haben das Präfix `bo_`.

### 2. Ersten Request senden

```bash
curl -X GET "https://your-instance.briefops.io/api/v1/meetings" \
  -H "X-API-Key: bo_<your_key>"
```

### 3. Meeting aus Transkript erstellen

```bash
curl -X POST "https://your-instance.briefops.io/api/v1/meetings" \
  -H "X-API-Key: bo_<your_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "Alice: Wir müssen das Deployment bis Freitag abschließen. Bob: Ich übernehme das.",
    "org_id": "org_abc123",
    "language": "de"
  }'
```

### 4. Tasks des Meetings abrufen

```bash
curl "https://your-instance.briefops.io/api/v1/tasks?meeting_id=mtg_abc123" \
  -H "X-API-Key: bo_<your_key>"
```

---

## Ressourcen

| Ressource | Beschreibung |
|---|---|
| [Authentifizierung](authentication.md) | API-Keys, Scopes, Rate Limits |
| [Endpoints](endpoints.md) | Alle Endpoints mit Request/Response-Beispielen |
| [Webhooks](webhooks.md) | Events, Payloads, Signaturprüfung |
| [SDKs](sdks.md) | Python & TypeScript SDK Installation und Nutzung |

---

## Allgemeine Konventionen

### Request-Format

- Alle Request-Bodies werden als `application/json` gesendet
- Zeitstempel folgen ISO 8601 (UTC): `2026-04-13T10:00:00Z`
- IDs haben Ressourcen-Präfixe: `mtg_`, `task_`, `org_`

### Response-Format

Alle Antworten enthalten API-Versionierung und Zeitstempel:

```json
{
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00",
  "data": { ... }
}
```

### Paginierung

Listen-Endpoints nutzen Cursor-basierte Paginierung:

```json
{
  "data": [...],
  "pagination": {
    "cursor": null,
    "next_cursor": "MjA=",
    "has_more": true,
    "page_size": 20,
    "total_count": 157
  }
}
```

Nächste Seite abrufen: `?cursor=MjA=`

### Fehlerformat

```json
{
  "error": "Meeting mtg_xyz not found",
  "code": "NOT_FOUND",
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00"
}
```

### HTTP-Statuscodes

| Code | Bedeutung |
|---|---|
| `200` | Erfolg |
| `201` | Ressource erstellt |
| `202` | Asynchroner Job angenommen |
| `204` | Kein Inhalt |
| `400` | Ungültige Anfrage |
| `401` | Authentifizierung fehlgeschlagen |
| `403` | Zugriff verweigert |
| `404` | Ressource nicht gefunden |
| `422` | Validierungsfehler |
| `429` | Rate Limit erreicht |
| `500` | Serverfehler |
