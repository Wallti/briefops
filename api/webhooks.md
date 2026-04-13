# Webhooks

BriefOps kann bei bestimmten Ereignissen HTTP-Callbacks an eine URL deiner Wahl senden. Dies ermöglicht Echtzeit-Integration mit externen Systemen wie Zapier, Make (ehem. Integromat) oder eigenen Backends.

---

## Webhook registrieren

```bash
curl -X POST "https://your-instance.briefops.io/api/v1/webhooks" \
  -H "X-API-Key: bo_<your_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://my-app.example.com/webhooks/briefops",
    "events": ["meeting.created", "task.completed"],
    "org_id": "org_abc"
  }'
```

**Response `201`:**
```json
{
  "id": "wh_5e3c1a2b",
  "org_id": "org_abc",
  "url": "https://my-app.example.com/webhooks/briefops",
  "events": ["meeting.created", "task.completed"],
  "secret": "whsec_f3a8b2c1d4e7f0a9b3c5d8e1f4a7b0c2",
  "enabled": true,
  "created_at": "2026-04-13T10:00:00Z"
}
```

Das `secret` wird zum Verifizieren der Signatur verwendet — **einmalig sichern**.

---

## Unterstützte Events

| Event | Auslöser |
|---|---|
| `meeting.created` | Neues Meeting wurde erstellt (via API oder Import) |
| `meeting.completed` | Meeting als abgeschlossen markiert |
| `task.created` | Task aus Meeting-Zusammenfassung extrahiert |
| `task.updated` | Task-Felder wurden geändert (Status, Owner, etc.) |
| `task.completed` | Task-Status auf `done` gesetzt |
| `summary.generated` | KI-Zusammenfassung eines Meetings fertiggestellt |
| `conflict.detected` | KI hat einen Termin- oder Ressourcenkonflikt erkannt |

---

## Payload-Format

Alle Webhook-Anfragen sind `POST`-Requests mit `Content-Type: application/json`.

### Gemeinsame Felder

```json
{
  "id": "evt_7a3c2b1d",
  "event": "meeting.created",
  "org_id": "org_abc",
  "timestamp": "2026-04-13T10:00:00Z",
  "data": { ... }
}
```

### `meeting.created`

```json
{
  "id": "evt_7a3c2b1d",
  "event": "meeting.created",
  "org_id": "org_abc",
  "timestamp": "2026-04-13T10:00:00Z",
  "data": {
    "meeting_id": "mtg_3f8a2c1d",
    "title": "Sprint Planning Q2",
    "language": "de",
    "created_by": "user_xyz",
    "created_at": "2026-04-13T10:00:00Z"
  }
}
```

### `task.created` / `task.updated`

```json
{
  "id": "evt_4b2d1c3e",
  "event": "task.created",
  "org_id": "org_abc",
  "timestamp": "2026-04-13T10:00:05Z",
  "data": {
    "task_id": "task_7c2b1a3e",
    "meeting_id": "mtg_3f8a2c1d",
    "title": "CI-Setup abschließen",
    "owner": "bob@example.com",
    "status": "open",
    "priority": "high",
    "deadline": "2026-04-18"
  }
}
```

### `task.completed`

```json
{
  "id": "evt_9f1e3a2c",
  "event": "task.completed",
  "org_id": "org_abc",
  "timestamp": "2026-04-18T14:30:00Z",
  "data": {
    "task_id": "task_7c2b1a3e",
    "title": "CI-Setup abschließen",
    "owner": "bob@example.com",
    "completed_at": "2026-04-18T14:30:00Z",
    "cycle_time_days": 5
  }
}
```

### `summary.generated`

```json
{
  "id": "evt_2c4a1b3d",
  "event": "summary.generated",
  "org_id": "org_abc",
  "timestamp": "2026-04-13T10:01:00Z",
  "data": {
    "meeting_id": "mtg_3f8a2c1d",
    "summary_text": "Das Team plant das Deployment...",
    "action_item_count": 3,
    "key_decision_count": 1
  }
}
```

### `conflict.detected`

```json
{
  "id": "evt_6d8b2a4c",
  "event": "conflict.detected",
  "org_id": "org_abc",
  "timestamp": "2026-04-13T10:02:00Z",
  "data": {
    "conflict_type": "deadline_overlap",
    "severity": "high",
    "affected_tasks": ["task_7c2b1a3e", "task_8d3c2b4f"],
    "description": "Zwei High-Priority-Tasks mit gleicher Deadline und gleichem Owner"
  }
}
```

---

## Signaturprüfung

Jede Webhook-Anfrage enthält einen `X-BriefOps-Signature`-Header mit einem HMAC-SHA256-Hash des Request-Bodys.

### Header-Format

```
X-BriefOps-Signature: sha256=a3f8c2d1e4b7f9a0...
X-BriefOps-Timestamp: 1744538400
```

### Verifikation in Python

```python
import hashlib
import hmac
import time

def verify_webhook(payload: bytes, signature: str, timestamp: str, secret: str) -> bool:
    # Replay-Schutz: Timestamp darf max. 5 Minuten alt sein
    if abs(time.time() - int(timestamp)) > 300:
        return False

    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256,
    ).hexdigest()

    received = signature.removeprefix("sha256=")
    return hmac.compare_digest(expected, received)


# FastAPI-Beispiel
from fastapi import Request, HTTPException

@app.post("/webhooks/briefops")
async def receive_webhook(request: Request):
    payload = await request.body()
    signature = request.headers.get("X-BriefOps-Signature", "")
    timestamp = request.headers.get("X-BriefOps-Timestamp", "")

    if not verify_webhook(payload, signature, timestamp, WEBHOOK_SECRET):
        raise HTTPException(status_code=401, detail="Invalid signature")

    event = await request.json()
    # Verarbeitung ...
    return {"ok": True}
```

### Verifikation in TypeScript

```typescript
import crypto from "crypto";

function verifyWebhook(
  payload: Buffer,
  signature: string,
  timestamp: string,
  secret: string
): boolean {
  // Replay-Schutz
  if (Math.abs(Date.now() / 1000 - parseInt(timestamp)) > 300) {
    return false;
  }

  const expected = crypto
    .createHmac("sha256", secret)
    .update(payload)
    .digest("hex");

  const received = signature.replace("sha256=", "");
  return crypto.timingSafeEqual(
    Buffer.from(expected, "hex"),
    Buffer.from(received, "hex")
  );
}
```

---

## Retry-Logik

BriefOps versucht fehlgeschlagene Webhook-Deliveries automatisch erneut:

| Versuch | Wartezeit |
|---|---|
| 1 (initial) | sofort |
| 2 | 30 Sekunden |
| 3 | 5 Minuten |

Als fehlgeschlagen gilt ein Delivery, wenn:
- Die Ziel-URL nicht erreichbar ist (Timeout: 10 Sekunden)
- HTTP-Statuscode ≥ 400 zurückgegeben wird

Nach 3 erfolglosen Versuchen wird das Event als `failed` markiert und nicht weiter zugestellt.

**Empfehlung:** Endpoint gibt `200 OK` zurück, sobald der Payload empfangen wurde — aufwendige Verarbeitung asynchron durchführen.

---

## Webhook verwalten

```bash
# Alle Webhooks einer Organisation
curl "https://your-instance.briefops.io/api/v1/webhooks?org_id=org_abc" \
  -H "X-API-Key: bo_<your_key>"

# Webhook deaktivieren
curl -X PATCH "https://your-instance.briefops.io/api/v1/webhooks/wh_5e3c1a2b" \
  -H "X-API-Key: bo_<your_key>" \
  -d '{"enabled": false}'

# Webhook löschen
curl -X DELETE "https://your-instance.briefops.io/api/v1/webhooks/wh_5e3c1a2b" \
  -H "X-API-Key: bo_<your_key>"
```
