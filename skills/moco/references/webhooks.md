# MOCO Webhooks — Referenz

## Übersicht

Webhooks senden HTTP POST Requests an eine konfigurierte URL, wenn in MOCO bestimmte Events auftreten.

## Setup

### Webhook erstellen (API)
```
POST /api/v1/account/web_hooks
{
  "target": "Activity",
  "event": "create",
  "hook": "https://your-endpoint.example.com/webhook"
}
```

### Webhook erstellen (UI)
Einstellungen → Erweiterungen → API & Webhooks → Webhook hinzufügen

---

## Targets & Events

Quelle: OpenAPI-Spec `https://docs.mocoapp.com/api/docs/v1.yaml` (Enum `target` bei `POST /account/web_hooks`).

| Target | Beschreibung |
|--------|-------------|
| `Activity` | Zeiteinträge |
| `Presence` | Arbeitszeiten |
| `Schedule` | Absenzen |
| `User` | Mitarbeitende |
| `User::HolidayRequest` | Urlaubsanträge |
| `User::AbsenceRequest` | Abwesenheitsanträge |
| `User::PeriodLockOverride` | Freigaben gesperrter Zeiträume |
| `User::WorkTimeAdjustment` | Arbeitszeit-Korrekturen |
| `Project` | Projekte |
| `ProjectGroup` | Projektgruppen |
| `Task` | Leistungen |
| `Contract` | Projekt-Zuweisungen |
| `Expense` | Zusatzleistungen |
| `RecurringExpense` | Wiederkehrende Zusatzleistungen |
| `Project::PaymentSchedule` | Zahlungspläne |
| `PlanningEntry` | Planungseinträge |
| `Company` | Firmen |
| `Contact` | Kontakte |
| `Deal` | Leads |
| `Comment` | Notizen |
| `Offer` | Angebote |
| `Invoice` | Rechnungen |
| `InvoiceDraft` | Rechnungsentwürfe |
| `InvoiceReminder` | Mahnungen |
| `Invoice::Approval` | Rechnungsfreigaben |
| `Invoice::Payment` | Zahlungseingänge |
| `Invoice::BookkeepingExport` | Buchhaltungsexporte (Rechnungen) |
| `Purchase` | Ausgaben |
| `Purchase::Draft` | Ausgaben-Entwürfe |
| `Purchase::Order` | Bestellungen |
| `Purchase::Approval` | Ausgaben-Freigaben |
| `Purchase::Payment` | Zahlungsausgänge |
| `Purchase::BookkeepingExport` | Buchhaltungsexporte (Ausgaben) |

Jedes Target unterstützt die Events: `create`, `update`, `delete`.

**Hinweise:**
- Zustellung ist nicht in Reihenfolge garantiert — `X-Moco-Timestamp` beachten.
- Webhooks stehen nur zahlenden Accounts zur Verfügung.
- Nur das Feld `hook` lässt sich nachträglich ändern (`PUT`/`PATCH /account/web_hooks/{id}`).

---

## Webhook-Payload

Der Body enthält das vollständige Objekt im gleichen Format wie der GET-Endpoint der API.

### Headers

| Header | Beschreibung |
|--------|-------------|
| `X-Moco-Target` | Target-Typ, z.B. "Activity" |
| `X-Moco-Event` | Event-Typ, z.B. "create" |
| `X-Moco-Timestamp` | Unix Timestamp des Events |
| `X-Moco-Signature` | HMAC SHA256 Signatur |
| `X-Moco-User-Id` | ID des Users der die Aktion ausgelöst hat |
| `X-Moco-Account-Url` | Account-URL, z.B. "https://firma.mocoapp.com" |

---

## Signatur-Verifizierung

Die `X-Moco-Signature` ist ein HMAC SHA256 Hash des Request-Body. Schlüssel ist der 32-stellige Hex-String aus der Webhook-Übersicht in MOCO — als Secret speichern, nie im Code oder Repo.

### Node.js Beispiel
```javascript
const crypto = require('crypto');

function verifySignature(body, signature, secret) {
  const hash = crypto
    .createHmac('sha256', secret)
    .update(body, 'utf8')
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(hash),
    Buffer.from(signature)
  );
}
```

### Python Beispiel
```python
import hmac
import hashlib

def verify_signature(body: bytes, signature: str, secret: str) -> bool:
    hash = hmac.new(
        secret.encode('utf-8'),
        body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(hash, signature)
```

---

## Wichtige Verhaltensregeln

1. **Timeout**: Muss innerhalb von **10 Sekunden** mit HTTP 2XX antworten
2. **Auto-Disable**: Nach **500 aufeinanderfolgenden Fehlern** wird der Webhook automatisch deaktiviert
3. **Reaktivierung**: `PUT /api/v1/account/web_hooks/{id}/enable`
4. **Reihenfolge**: Events werden nicht garantiert in Reihenfolge zugestellt
5. **Duplikate**: Es kann zu doppelten Zustellungen kommen — Idempotenz im Empfänger sicherstellen

---

## Verwaltung

```
GET    /api/v1/account/web_hooks           — Alle Webhooks auflisten
GET    /api/v1/account/web_hooks/{id}      — Einzeln
PUT    /api/v1/account/web_hooks/{id}      — URL aktualisieren (nur hook-Feld)
PUT    /api/v1/account/web_hooks/{id}/enable   — Aktivieren
PUT    /api/v1/account/web_hooks/{id}/disable  — Deaktivieren
DELETE /api/v1/account/web_hooks/{id}      — Löschen
```

---

## Best Practices

- **Schnell antworten**: Schwere Verarbeitung asynchron auslagern (Queue, Worker)
- **Idempotenz**: Gleiche Events mehrfach verarbeiten können
- **Signatur prüfen**: Immer `X-Moco-Signature` verifizieren
- **Monitoring**: Webhook-Fehler überwachen, bevor 500-Fehler-Limit erreicht wird
- **Retry-Logik**: MOCO hat kein eingebautes Retry — bei Fehler geht das Event verloren
