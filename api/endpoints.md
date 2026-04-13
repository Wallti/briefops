# API Endpoints

Alle Endpoints sind unter `/api/v1` erreichbar. Authentifizierung via `X-API-Key` Header ist Pflicht (außer `/health`).

**OpenAPI Spec:** `GET /api/v1/openapi.json`

---

## Health

### `GET /health`

Gibt den API-Status zurück. Keine Authentifizierung erforderlich.

**Response `200`:**
```json
{
  "status": "healthy",
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00"
}
```

---

## Meetings

### `GET /meetings`

Listet alle Meetings auf (cursor-basiert paginiert).

**Query-Parameter:**

| Parameter | Typ | Default | Beschreibung |
|---|---|---|---|
| `cursor` | string | — | Paginierungs-Cursor aus vorheriger Antwort |
| `page_size` | integer | `20` | Einträge pro Seite (max. `100`) |
| `sort_by` | string | `created_at` | Sortierfeld: `created_at`, `updated_at` |
| `sort_order` | string | `desc` | `asc` oder `desc` |
| `org_id` | string | — | Filter nach Organisation |

**Request:**
```bash
curl "https://your-instance.briefops.io/api/v1/meetings?page_size=10&org_id=org_abc" \
  -H "X-API-Key: bo_<your_key>"
```

**Response `200`:**
```json
{
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00",
  "data": [
    {
      "id": "mtg_3f8a2c1d",
      "title": "Sprint Planning Q2",
      "org_id": "org_abc",
      "language": "de",
      "created_at": "2026-04-13T09:00:00Z",
      "updated_at": "2026-04-13T09:05:00Z",
      "created_by": "user_xyz"
    }
  ],
  "pagination": {
    "cursor": null,
    "next_cursor": "MTA=",
    "has_more": true,
    "page_size": 10,
    "total_count": 42
  }
}
```

---

### `POST /meetings`

Erstellt ein neues Meeting aus einem Transkript. Die API generiert automatisch eine KI-Zusammenfassung und extrahiert Tasks.

**Request-Body:**

| Feld | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `transcript` | string | ja | Rohtext (10–100.000 Zeichen) |
| `org_id` | string | ja | Organisations-ID |
| `title` | string | nein | Titel (wird sonst aus Summary generiert) |
| `language` | string | nein | Sprache (`de` default, `en` möglich) |

**Request:**
```bash
curl -X POST "https://your-instance.briefops.io/api/v1/meetings" \
  -H "X-API-Key: bo_<your_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "Alice: Wir müssen das Deployment bis Freitag abschließen.\nBob: Ich übernehme das CI-Setup.",
    "org_id": "org_abc",
    "language": "de"
  }'
```

**Response `201`:**
```json
{
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00",
  "data": {
    "id": "mtg_3f8a2c1d",
    "title": "Deployment-Planung",
    "org_id": "org_abc",
    "transcript": "Alice: Wir müssen...",
    "language": "de",
    "summary": {
      "meeting_id": "mtg_3f8a2c1d",
      "summary_text": "Das Team plant das Deployment für Freitag.",
      "action_items": [
        {"description": "CI-Setup abschließen", "assignee": "Bob", "due_date": "2026-04-18"}
      ],
      "key_decisions": ["Deployment-Deadline: Freitag"]
    },
    "created_at": "2026-04-13T10:00:00Z",
    "updated_at": "2026-04-13T10:00:00Z",
    "created_by": "user_xyz"
  }
}
```

---

### `GET /meetings/{meeting_id}`

Gibt ein einzelnes Meeting zurück.

**Pfad-Parameter:**

| Parameter | Beschreibung |
|---|---|
| `meeting_id` | ID des Meetings (Präfix `mtg_`) |

**Request:**
```bash
curl "https://your-instance.briefops.io/api/v1/meetings/mtg_3f8a2c1d" \
  -H "X-API-Key: bo_<your_key>"
```

**Response `200`:** Wie `POST /meetings` → `data`-Objekt

**Response `404`:**
```json
{
  "error": "Meeting mtg_3f8a2c1d not found",
  "code": "NOT_FOUND"
}
```

---

## Tasks

### `GET /tasks`

Listet Tasks auf, optional gefiltert.

**Query-Parameter:**

| Parameter | Typ | Default | Beschreibung |
|---|---|---|---|
| `cursor` | string | — | Paginierungs-Cursor |
| `page_size` | integer | `20` | Max. `100` |
| `sort_by` | string | `created_at` | `created_at`, `deadline`, `priority` |
| `sort_order` | string | `desc` | `asc` oder `desc` |
| `org_id` | string | — | Filter nach Organisation |
| `status` | string | — | `open`, `in_progress`, `done`, `cancelled` |
| `priority` | string | — | `low`, `medium`, `high` |
| `owner` | string | — | User-ID oder E-Mail |
| `meeting_id` | string | — | Nur Tasks eines bestimmten Meetings |

**Request:**
```bash
curl "https://your-instance.briefops.io/api/v1/tasks?status=open&owner=bob@example.com" \
  -H "X-API-Key: bo_<your_key>"
```

**Response `200`:**
```json
{
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00",
  "data": [
    {
      "id": "task_7c2b1a3e",
      "meeting_id": "mtg_3f8a2c1d",
      "org_id": "org_abc",
      "title": "CI-Setup abschließen",
      "owner": "bob@example.com",
      "deadline": "2026-04-18",
      "priority": "high",
      "status": "open",
      "created_at": "2026-04-13T10:00:00Z",
      "updated_at": "2026-04-13T10:00:00Z"
    }
  ],
  "pagination": {
    "cursor": null,
    "next_cursor": null,
    "has_more": false,
    "page_size": 20,
    "total_count": 1
  }
}
```

---

### `GET /tasks/{task_id}`

Gibt einen einzelnen Task zurück.

**Request:**
```bash
curl "https://your-instance.briefops.io/api/v1/tasks/task_7c2b1a3e" \
  -H "X-API-Key: bo_<your_key>"
```

**Response `200`:** Task-Objekt (wie oben in `data`-Array)

---

### `PATCH /tasks/{task_id}`

Aktualisiert einen Task. Nur übergebene Felder werden geändert.

**Request-Body** (alle Felder optional):

| Feld | Typ | Beschreibung |
|---|---|---|
| `title` | string | Neuer Titel |
| `owner` | string | Neuer Besitzer |
| `status` | string | `open`, `in_progress`, `done`, `cancelled` |
| `priority` | string | `low`, `medium`, `high` |
| `deadline` | string | ISO-Datum `YYYY-MM-DD` |

**Request:**
```bash
curl -X PATCH "https://your-instance.briefops.io/api/v1/tasks/task_7c2b1a3e" \
  -H "X-API-Key: bo_<your_key>" \
  -H "Content-Type: application/json" \
  -d '{"status": "done"}'
```

**Response `200`:** Aktualisiertes Task-Objekt

---

## Summaries

### `GET /summaries/{meeting_id}`

Gibt die KI-generierte Zusammenfassung eines Meetings zurück.

**Request:**
```bash
curl "https://your-instance.briefops.io/api/v1/summaries/mtg_3f8a2c1d" \
  -H "X-API-Key: bo_<your_key>"
```

**Response `200`:**
```json
{
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00",
  "data": {
    "meeting_id": "mtg_3f8a2c1d",
    "summary_text": "Das Team bespricht das Deployment für Freitag. Bob übernimmt CI-Setup.",
    "action_items": [
      {
        "description": "CI-Setup abschließen",
        "assignee": "Bob",
        "due_date": "2026-04-18",
        "priority": "high"
      }
    ],
    "key_decisions": [
      "Deployment-Deadline: Freitag, 18. April"
    ],
    "participants": ["Alice", "Bob"],
    "duration_minutes": 30
  }
}
```

---

## Transkription

### `POST /transcribe`

Sendet einen Audio-Transkriptions-Job (asynchron). Die Antwort enthält eine Job-ID zum Statusabruf.

**Request-Body:**

| Feld | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `audio_url` | string | ja | URL zur Audio-Datei |
| `language` | string | nein | Sprache (`de` default) |
| `callback_url` | string | nein | Webhook-URL für Benachrichtigung bei Fertigstellung |

**Request:**
```bash
curl -X POST "https://your-instance.briefops.io/api/v1/transcribe" \
  -H "X-API-Key: bo_<your_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "audio_url": "https://storage.example.com/meeting-2026-04-13.mp3",
    "language": "de",
    "callback_url": "https://my-app.example.com/webhooks/briefops"
  }'
```

**Response `202`:**
```json
{
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00",
  "job_id": "job_9e4d2f1a",
  "status": "queued"
}
```

**Job-Status-Werte:** `queued` → `processing` → `completed` | `failed`

---

## Organisations-Insights

### `GET /organizations/{org_id}/insights`

Gibt Analytics und Performance-Metriken für eine Organisation zurück.

**Query-Parameter:**

| Parameter | Typ | Default | Beschreibung |
|---|---|---|---|
| `period` | string | `monthly` | `weekly`, `monthly`, `quarterly` |

**Request:**
```bash
curl "https://your-instance.briefops.io/api/v1/organizations/org_abc/insights?period=weekly" \
  -H "X-API-Key: bo_<your_key>"
```

**Response `200`:**
```json
{
  "api_version": "1.0.0",
  "timestamp": "2026-04-13T10:00:00.000000+00:00",
  "org_id": "org_abc",
  "meeting_stats": {
    "total_meetings": 48,
    "avg_duration_minutes": 34,
    "meetings_per_week": 6
  },
  "velocity": {
    "tasks_created": 120,
    "tasks_completed": 98,
    "completion_rate": 0.82
  },
  "team_performance": {
    "top_contributors": ["Alice", "Bob"],
    "avg_task_cycle_time_days": 2.4
  },
  "overdue_trends": {
    "overdue_count": 7,
    "overdue_rate": 0.07,
    "trend": "improving"
  }
}
```
