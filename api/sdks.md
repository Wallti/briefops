# SDKs

BriefOps bietet offizielle Client-Bibliotheken für Python und TypeScript.

---

## Python SDK

### Installation

```bash
pip install briefops
# oder mit poetry
poetry add briefops
```

Voraussetzungen: Python 3.10+, `httpx` (wird automatisch installiert)

### Konfiguration

```python
from briefops.sdk import BriefOpsClient

client = BriefOpsClient(
    api_key="bo_<your_key>",          # Pflicht
    base_url="https://your-instance.briefops.io",  # Default: http://localhost:8000
    timeout=30.0,                      # Sekunden, Default: 30
)
```

**Via Umgebungsvariable (empfohlen):**

```python
import os
from briefops.sdk import BriefOpsClient

client = BriefOpsClient(api_key=os.environ["BRIEFOPS_API_KEY"])
```

### Context Manager

```python
with BriefOpsClient(api_key="bo_...") as client:
    meetings = client.list_meetings()
# HTTP-Verbindungen werden automatisch geschlossen
```

---

### Meetings

#### Meetings auflisten

```python
# Erste Seite
result = client.list_meetings(page_size=10)
meetings = result["data"]
next_cursor = result["pagination"]["next_cursor"]

# Nächste Seite
if next_cursor:
    result2 = client.list_meetings(page_size=10, cursor=next_cursor)

# Alle Meetings iterieren
cursor = None
while True:
    result = client.list_meetings(page_size=50, cursor=cursor)
    for meeting in result["data"]:
        process(meeting)
    cursor = result["pagination"]["next_cursor"]
    if not cursor:
        break
```

#### Einzelnes Meeting abrufen

```python
meeting = client.get_meeting("mtg_3f8a2c1d")
print(meeting["data"]["title"])
```

#### Meeting aus Transkript erstellen

```python
result = client.create_meeting(
    transcript="Alice: Wir müssen das Deployment bis Freitag abschließen.\n"
               "Bob: Ich übernehme das CI-Setup.",
    org_id="org_abc",
)
meeting = result["data"]
print(f"Meeting erstellt: {meeting['id']}")
print(f"Extrahierte Tasks: {len(meeting['summary']['action_items'])}")
```

---

### Tasks

#### Tasks auflisten

```python
# Alle offenen Tasks
result = client.list_tasks(status="open")

# Tasks eines bestimmten Owners
result = client.list_tasks(owner="bob@example.com")

# Kombinierte Filter
result = client.list_tasks(status="open", owner="alice@example.com")
```

#### Task abrufen

```python
task = client.get_task("task_7c2b1a3e")
print(task["data"]["title"])
```

#### Task aktualisieren

```python
# Status setzen
updated = client.update_task("task_7c2b1a3e", status="done")

# Mehrere Felder
updated = client.update_task(
    "task_7c2b1a3e",
    status="in_progress",
    owner="carol@example.com",
    priority="high",
)
```

---

### Summaries

```python
summary = client.get_summary("mtg_3f8a2c1d")
data = summary["data"]
print(data["summary_text"])
for item in data["action_items"]:
    print(f"- {item['description']} ({item['assignee']})")
```

---

### Transkription

```python
# Lokale Audio-Datei transkribieren
with open("meeting.mp3", "rb") as f:
    audio_bytes = f.read()

result = client.transcribe(audio_bytes, format="mp3")
print(result["transcript"])
print(f"Dauer: {result['duration_seconds']}s, Sprache: {result['language']}")
```

Unterstützte Formate: `mp3`, `wav`, `m4a`, `ogg`, `webm`

---

### Fehlerbehandlung

```python
from briefops.sdk import (
    BriefOpsClient,
    BriefOpsError,
    BriefOpsAuthError,
    BriefOpsNotFoundError,
    BriefOpsRateLimitError,
)
import time

client = BriefOpsClient(api_key="bo_...")

try:
    meeting = client.get_meeting("mtg_unknown")

except BriefOpsAuthError as e:
    print(f"Authentifizierung fehlgeschlagen: {e}")
    # Key prüfen oder erneuern

except BriefOpsNotFoundError as e:
    print(f"Nicht gefunden: {e}")

except BriefOpsRateLimitError as e:
    print(f"Rate Limit erreicht. Retry nach {e.retry_after}s")
    time.sleep(e.retry_after or 5)

except BriefOpsError as e:
    print(f"API-Fehler {e.status_code}: {e}")
```

---

## TypeScript SDK

### Installation

```bash
npm install @briefops/sdk
# oder
yarn add @briefops/sdk
# oder
pnpm add @briefops/sdk
```

Voraussetzungen: Node.js 18+

### Konfiguration

```typescript
import { BriefOpsClient } from "@briefops/sdk";

const client = new BriefOpsClient({
  apiKey: process.env.BRIEFOPS_API_KEY!,
  baseUrl: "https://your-instance.briefops.io", // optional
  timeout: 30_000, // ms, optional
});
```

---

### Meetings

```typescript
// Meetings auflisten
const result = await client.listMeetings({ pageSize: 10 });
const meetings = result.data;
const nextCursor = result.pagination.nextCursor;

// Meeting erstellen
const created = await client.createMeeting({
  transcript:
    "Alice: Wir müssen das Deployment bis Freitag abschließen.\nBob: Ich übernehme das.",
  orgId: "org_abc",
  language: "de",
});
console.log(`Meeting erstellt: ${created.data.id}`);

// Meeting abrufen
const meeting = await client.getMeeting("mtg_3f8a2c1d");
```

---

### Tasks

```typescript
// Tasks auflisten
const tasks = await client.listTasks({
  status: "open",
  owner: "bob@example.com",
});

// Task aktualisieren
const updated = await client.updateTask("task_7c2b1a3e", {
  status: "done",
});

// Summary abrufen
const summary = await client.getSummary("mtg_3f8a2c1d");
console.log(summary.data.summaryText);
```

---

### Transkription

```typescript
import { readFileSync } from "fs";

const audioBuffer = readFileSync("meeting.mp3");
const result = await client.transcribe(audioBuffer, { format: "mp3" });
console.log(result.transcript);
```

---

### Fehlerbehandlung

```typescript
import {
  BriefOpsError,
  BriefOpsAuthError,
  BriefOpsNotFoundError,
  BriefOpsRateLimitError,
} from "@briefops/sdk";

try {
  const meeting = await client.getMeeting("mtg_unknown");
} catch (e) {
  if (e instanceof BriefOpsAuthError) {
    console.error("Authentifizierung fehlgeschlagen:", e.message);
  } else if (e instanceof BriefOpsNotFoundError) {
    console.error("Nicht gefunden:", e.message);
  } else if (e instanceof BriefOpsRateLimitError) {
    console.error(`Rate Limit. Retry nach ${e.retryAfter}s`);
    await new Promise((r) => setTimeout(r, (e.retryAfter ?? 5) * 1000));
  } else if (e instanceof BriefOpsError) {
    console.error(`API-Fehler ${e.statusCode}:`, e.message);
  }
}
```

---

### TypeScript-Typen

Das SDK exportiert vollständige TypeScript-Typen:

```typescript
import type {
  Meeting,
  Task,
  MeetingSummary,
  PaginationMeta,
  TaskStatus,
  Priority,
  WebhookEvent,
} from "@briefops/sdk";

// Beispiel
const handleTask = (task: Task): void => {
  if (task.status === "open" && task.priority === "high") {
    // ...
  }
};
```

---

## Weitere Ressourcen

- [API Endpoints Referenz](endpoints.md)
- [Authentifizierung & Rate Limits](authentication.md)
- [Webhooks](webhooks.md)
- GitHub: `briefops/briefops-python-sdk` | `briefops/briefops-js-sdk`
- Changelog: `CHANGELOG.md` im jeweiligen SDK-Repository
