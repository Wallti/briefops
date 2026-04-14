# Authentifizierung

Die BriefOps API verwendet API-Keys zur Authentifizierung. Alle Requests (außer `/health`) müssen einen gültigen Key mitschicken.

---

## API-Keys

### Format

Alle Keys haben das Präfix `bo_` gefolgt von 64 zufälligen Hex-Zeichen:

```
bo_a3f8c2d1e4b7f9a0c3d5e8f1a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2e4f6a8
```

Der vollständige Key wird **nur einmal** bei der Erstellung angezeigt. Danach wird nur noch der SHA256-Hash gespeichert.

### Key erstellen

Im BriefOps-Dashboard: **Settings → API Keys → New Key**

Oder über die Management-API (erfordert Admin-Scope):

```bash
curl -X POST "https://your-instance.briefops.io/api/v1/api-keys" \
  -H "X-API-Key: bo_<admin_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CI/CD Key",
    "scopes": ["read", "write"]
  }'
```

**Response:**
```json
{
  "key": "bo_a3f8c2...",
  "id": "key_abc123",
  "name": "CI/CD Key",
  "scopes": ["read", "write"],
  "created_at": "2026-04-13T10:00:00Z"
}
```

### Key verwenden

Der Key wird im `X-API-Key` Header übertragen:

```bash
curl -H "X-API-Key: bo_<your_key>" \
  "https://your-instance.briefops.io/api/v1/meetings"
```

Mit dem Python SDK:

```python
from briefops.sdk import BriefOpsClient

client = BriefOpsClient(api_key="bo_<your_key>")
```

### Key widerrufen

```bash
curl -X DELETE "https://your-instance.briefops.io/api/v1/api-keys/key_abc123" \
  -H "X-API-Key: bo_<admin_key>"
```

---

## Scopes

Scopes steuern den Zugriff auf API-Ressourcen:

| Scope | Beschreibung |
|---|---|
| `read` | Lesezugriff auf Meetings, Tasks, Summaries |
| `write` | Schreibzugriff (erstellen, aktualisieren) |
| `transcribe` | Zugriff auf Transkriptions-Endpoint |
| `insights` | Zugriff auf Organisations-Analytics |
| `admin` | Vollzugriff inkl. Key-Verwaltung |

---

## Fehlerbehandlung

### 401 Unauthorized

```json
{
  "error": "Invalid API key",
  "code": "UNAUTHORIZED"
}
```

Ursachen:
- Key fehlt im Header
- Key ungültig oder widerrufen
- Key abgelaufen (falls Ablaufdatum gesetzt)

### 403 Forbidden

```json
{
  "error": "Insufficient scopes",
  "code": "FORBIDDEN"
}
```

Der Key ist gültig, hat aber nicht den nötigen Scope für die angeforderte Operation.

---

## Rate Limits

Die API verwendet einen **Token Bucket Algorithmus** mit per-Key- und per-Endpoint-Limits.

### Standard-Limits

| Endpoint-Gruppe | Requests/Minute |
|---|---|
| `/v1/meetings` | 60 |
| `/v1/tasks` | 60 |
| `/v1/summaries` | 60 |
| `/v1/transcribe` | 10 |
| `/v1/organizations/*/insights` | 20 |
| `/v1/health` | unbegrenzt |

### Rate-Limit-Header

Jede Antwort enthält aktuelle Limit-Informationen:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1744543260
```

### 429 Too Many Requests

```json
{
  "error": "Rate limit exceeded",
  "code": "RATE_LIMIT_EXCEEDED"
}
```

Mit `Retry-After`-Header (Sekunden bis zum nächsten erlaubten Request):

```
Retry-After: 15
```

**Empfohlenes Verhalten:** Exponentielles Backoff mit Jitter bei `429`-Antworten.

```python
import time, random

def request_with_backoff(client, fn, *args, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fn(*args)
        except BriefOpsRateLimitError as e:
            if attempt == max_retries - 1:
                raise
            wait = (e.retry_after or 2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait)
```

---

## Sicherheitshinweise

- **HTTPS erforderlich** — alle Requests müssen über TLS laufen
- **Keys nicht loggen** — API-Keys niemals in Logs, Git oder Client-Code speichern
- **Minimale Scopes** — nur die wirklich benötigten Scopes vergeben
- **Key-Rotation** — Keys regelmäßig erneuern und alte widerrufen
- **Umgebungsvariablen** — Keys über Umgebungsvariablen injizieren, nicht hardcoden

```bash
# .env
BRIEFOPS_API_KEY=bo_<your_key>
```

```python
import os
from briefops.sdk import BriefOpsClient

client = BriefOpsClient(api_key=os.environ["BRIEFOPS_API_KEY"])
```
