# Dream Notes n8n Workflows

Dieses Repository enthält zwei n8n-Workflows für eine kleine Dream-Notes-App:

```text
MainWorkflow.json   # WhatsApp → Audio → Transkript → KI-Analyse → Data Table → Bildgenerierung
APIs.json           # REST-API für das Frontend: Notizen lesen, erstellen, bearbeiten, löschen
```

Der Fokus dieser README liegt nicht auf der internen Logik jedes Nodes, sondern darauf, die Workflows möglichst schnell in n8n zu importieren, mit Credentials zu verbinden und vom Frontend aus nutzbar zu machen.

---

## 1. Architektur in Kurzform

```text
WhatsApp-Sprachnachricht
        ↓
MainWorkflow.json
        ↓
n8n Data Table: voice_notes_table
        ↓
APIs.json
        ↓
Frontend
```

`MainWorkflow.json` verarbeitet WhatsApp-Audios: Audio herunterladen, transkribieren, mit Gemini strukturieren, in einer n8n Data Table speichern und optional ein Bild generieren.

`APIs.json` stellt die gespeicherten Dream Notes für ein Frontend bereit. Das Frontend spricht nur diesen Workflow an:

```text
/webhook/notes
```

---

## 2. Voraussetzungen

Du brauchst:

| Dienst | Wofür? |
|---|---|
| n8n | Workflow-Ausführung und Data Table |
| Twilio WhatsApp | Empfang und Versand von WhatsApp-Nachrichten |
| AssemblyAI | Transkription der Audiodateien |
| Google Gemini API | JSON-Analyse des Traums und Bildgenerierung |
| S3-kompatibler Speicher, z. B. Cloudflare R2 | Speicherung der generierten Bilder |
| Frontend | Ruft die Notes-API auf |

Zusätzlich muss n8n von außen erreichbar sein, idealerweise über HTTPS, damit Twilio und das Frontend die Webhooks erreichen können.

---

## 3. Workflows in n8n importieren

1. n8n öffnen.
2. Neuen Workflow erstellen.
3. `Import from File` oder `Import from JSON` auswählen.
4. Zuerst `MainWorkflow.json` importieren.
5. Danach `APIs.json` importieren.
6. Beide Workflows zunächst deaktiviert lassen, bis alle Credentials und Tabellen korrekt gesetzt sind.

Nach dem Import enthalten die Nodes zwar Credential-Namen, aber nicht die echten Secrets. Die Credentials müssen in deiner n8n-Instanz neu angelegt oder den Nodes neu zugewiesen werden.

---

## 4. Data Table anlegen

Lege in n8n eine Data Table mit dem Namen an:

```text
voice_notes_table
```

Die Tabelle sollte folgende Spalten besitzen:

| Spalte | Typ |
|---|---|
| title | string |
| category | string |
| shortSummary | string |
| fullSummary | string |
| transcript | string |
| keyTopics | string |
| people | string |
| places | string |
| mood | string |
| actionItems | string |
| reflectionNotes | string |
| imageStatus | string |
| imageUrl | string |

Die Array-Felder werden in der Tabelle als String gespeichert. Im `MainWorkflow.json` werden sie mit `JSON.stringify(...)` gespeichert. Im `APIs.json` werden sie beim Auslesen wieder zu echten Arrays geparst.

Wichtig: Nach dem Import musst du in allen Data-Table-Nodes die neu angelegte Tabelle auswählen, weil die exportierte interne Table-ID aus einer anderen n8n-Instanz stammt.

Betroffene Nodes:

```text
MainWorkflow.json:
- Insert row
- Upsert row(s)

APIs.json:
- Get row(s)
- Insert row
- Update row(s)
- Delete row(s)
```

---

## 5. Credentials einrichten

### 5.1 Twilio API Credential

Wird für WhatsApp-Nachrichten verwendet.

In n8n anlegen:

```text
Credential type: Twilio API
Name: Twilio account
```

Danach dieses Credential in allen Twilio-Nodes auswählen.

Betroffene Nodes im Main Workflow:

```text
To WA: Sprachaufnahme erhalten
To WA: Media Fehler
To WA: Transkription fertig
Fehler: Gemini
To WA: Dein Traum wurde erstellt.
To WA: Bildgenerierung wird gestartet.1
```

Passe außerdem in diesen Nodes die Telefonnummern an:

```text
from: deine Twilio-WhatsApp-Nummer
to: deine Ziel-WhatsApp-Nummer
```

Beispiel:

```text
from: whatsapp:+14155238886
to: whatsapp:+49...
```

Je nach n8n/Twilio-Konfiguration kann das Feld auch mit `toWhatsapp: true` arbeiten. Entscheidend ist, dass Sender- und Empfängernummer zu deinem Twilio-Setup passen.

---

### 5.2 HTTP Basic Auth für Twilio-Media-Download

Der Node `HTTP Request` lädt die von Twilio bereitgestellte Audiodatei herunter.

In n8n anlegen:

```text
Credential type: HTTP Basic Auth
Name: Twilio Media Basic Auth
Username: TWILIO_ACCOUNT_SID
Password: TWILIO_AUTH_TOKEN
```

Dann im Node `HTTP Request` dieses Credential auswählen.

---

### 5.3 AssemblyAI Credential

Wird für die Transkription der Audiodatei verwendet.

In n8n anlegen:

```text
Credential type: AssemblyAI API
Name: AssemblyAI account
API Key: dein AssemblyAI API Key
```

Betroffene Nodes:

```text
Upload a file
Create a transcription
Get a transcription
```

Falls der AssemblyAI-Node in deiner n8n-Instanz fehlt, muss das entsprechende Community Node Package installiert werden.

---

### 5.4 Google Gemini Credential

Wird für die strukturierte Traum-Analyse und die Bildgenerierung verwendet.

In n8n anlegen:

```text
Credential type: Google Gemini / Google PaLM API
Name: Google Gemini API
API Key: dein Google AI Studio API Key
```

Betroffene Nodes:

```text
Message a model
Generate an image
```

Verwendete Modelle im Workflow:

```text
models/gemini-2.5-flash
models/gemini-2.5-flash-image
```

Falls diese Modelle in deiner Umgebung nicht verfügbar sind, wähle im jeweiligen Node ein verfügbares Gemini-Modell aus.

---

### 5.5 S3 / Cloudflare R2 Credential

Wird verwendet, um generierte Bilder hochzuladen.

In n8n anlegen:

```text
Credential type: S3
Name: S3 / R2 account
Access Key ID: dein Access Key
Secret Access Key: dein Secret
Endpoint: dein S3/R2 Endpoint
Region: auto oder passende Region
```

Betroffener Node:

```text
Upload a file1
```

Im Workflow ist als Bucket vorgesehen:

```text
dream-images
```

Passe außerdem im Node `Upsert row(s)` die öffentliche Bild-URL an:

```text
https://<deine-public-bucket-domain>/dream-{{ $('Insert row').item.json.id }}.png
```

---

### 5.6 Header Auth für die Frontend-API

Der `APIs.json`-Workflow ist über Header Auth geschützt.

In n8n anlegen:

```text
Credential type: Header Auth
Name: Header Auth account
Header Name: x-api-key
Header Value: ein langes geheimes Token
```

Das Frontend muss diesen Header bei jedem Request mitsenden:

```http
x-api-key: <DEIN_API_KEY>
```

Speichere den API-Key im Frontend nicht hart im Code, sondern über eine Umgebungsvariable.

---

## 6. Main Workflow: WhatsApp-Webhook einrichten

Der Main Workflow verwendet diesen Webhook:

```text
POST /webhook/whatsapp
```

Die vollständige Production-URL sieht typischerweise so aus:

```text
https://<deine-n8n-domain>/webhook/whatsapp
```

In Twilio musst du diese URL als Incoming-Message-Webhook für WhatsApp eintragen.

Erwartet wird ein Twilio-Webhook-Body mit unter anderem:

```text
MessageType=audio
MediaUrl0=<URL der Audiodatei>
```

Testablauf:

1. Workflow aktivieren.
2. Twilio WhatsApp Sandbox oder produktive WhatsApp-Nummer konfigurieren.
3. WhatsApp-Sprachnachricht an die Twilio-Nummer senden.
4. n8n sollte die Audiodatei herunterladen, transkribieren, analysieren und eine neue Zeile in `voice_notes_table` erstellen.
5. Danach sollte das Frontend die neue Notiz über die Notes-API lesen können.

---

## 7. Frontend-API

Das Frontend spricht den Workflow aus `APIs.json` an.

Basis-URL:

```text
https://<deine-n8n-domain>/webhook/notes
```

Für lokale Tests in n8n kann die Test-URL je nach n8n-Instanz so aussehen:

```text
https://<deine-n8n-domain>/webhook-test/notes
```

Jeder Request braucht den API-Key:

```http
x-api-key: <DEIN_API_KEY>
```

Für Requests mit Body zusätzlich:

```http
Content-Type: application/json
```

---

## 8. API-Endpunkte

### 8.1 Alle Notes abrufen

```http
GET /webhook/notes
```

Beispiel mit `fetch`:

```js
const response = await fetch(`${N8N_API_URL}/webhook/notes`, {
  method: "GET",
  headers: {
    "x-api-key": import.meta.env.VITE_N8N_API_KEY,
  },
});

const notes = await response.json();
```

Antwort:

```json
[
  {
    "id": 17,
    "title": "Unfall in Indoorspielplatz",
    "category": "Surrealer Traum",
    "shortSummary": "Ein Ausflug mit der Familie endet in einer absurden Indoor-Spiellandschaft.",
    "fullSummary": "Längere Zusammenfassung des Traums.",
    "transcript": "Vollständiges Transkript der Sprachnachricht.",
    "keyTopics": ["Indoorspielplatz", "Familie", "Unfall"],
    "people": ["Eltern", "Tobi", "Lucy"],
    "places": ["Urlaubsort", "Indoor-Spiellandschaft"],
    "mood": ["Verwirrung", "Stress"],
    "actionItems": [],
    "reflectionNotes": ["Möglicher Hinweis auf Kontrollverlust."],
    "imageStatus": "ready",
    "imageUrl": "https://<deine-public-bucket-domain>/dream-17.png"
  }
]
```

---

### 8.2 Note erstellen

```http
POST /webhook/notes
```

Request Body:

```json
{
  "title": "Unfall in Indoorspielplatz",
  "category": "Surrealer Traum",
  "shortSummary": "Ein Ausflug mit der Familie endet in einer absurden Indoor-Spiellandschaft.",
  "fullSummary": "Längere Zusammenfassung des Traums.",
  "transcript": "Vollständiges Transkript.",
  "keyTopics": ["Indoorspielplatz", "Familie", "Unfall"],
  "people": ["Eltern", "Tobi", "Lucy"],
  "places": ["Urlaubsort", "Indoor-Spiellandschaft"],
  "mood": ["Verwirrung", "Stress"],
  "actionItems": [],
  "reflectionNotes": ["Möglicher Hinweis auf Kontrollverlust."]
}
```

Beispiel mit `fetch`:

```js
const response = await fetch(`${N8N_API_URL}/webhook/notes`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "x-api-key": import.meta.env.VITE_N8N_API_KEY,
  },
  body: JSON.stringify(note),
});

const createdNote = await response.json();
```

Antwort:

```json
[
  {
    "id": 18,
    "title": "Unfall in Indoorspielplatz",
    "category": "Surrealer Traum",
    "shortSummary": "Ein Ausflug mit der Familie endet in einer absurden Indoor-Spiellandschaft.",
    "fullSummary": "Längere Zusammenfassung des Traums.",
    "transcript": "Vollständiges Transkript.",
    "keyTopics": ["Indoorspielplatz", "Familie", "Unfall"],
    "people": ["Eltern", "Tobi", "Lucy"],
    "places": ["Urlaubsort", "Indoor-Spiellandschaft"],
    "mood": ["Verwirrung", "Stress"],
    "actionItems": [],
    "reflectionNotes": ["Möglicher Hinweis auf Kontrollverlust."],
    "imageStatus": null,
    "imageUrl": null
  }
]
```

---

### 8.3 Note aktualisieren

```http
PATCH /webhook/notes?id=<NOTE_ID>
```

Beispiel:

```http
PATCH /webhook/notes?id=18
```

Request Body:

```json
{
  "title": "Geänderter Titel",
  "category": "Surrealer Traum",
  "shortSummary": "Neue Kurzbeschreibung.",
  "fullSummary": "Neue ausführliche Zusammenfassung.",
  "transcript": "Aktualisiertes Transkript.",
  "keyTopics": ["Thema 1", "Thema 2"],
  "people": ["Person 1"],
  "places": ["Ort 1"],
  "mood": ["Verwirrung"],
  "actionItems": [],
  "reflectionNotes": ["Neue Reflexion."]
}
```

Beispiel mit `fetch`:

```js
const response = await fetch(`${N8N_API_URL}/webhook/notes?id=${id}`, {
  method: "PATCH",
  headers: {
    "Content-Type": "application/json",
    "x-api-key": import.meta.env.VITE_N8N_API_KEY,
  },
  body: JSON.stringify(updatedNote),
});

const updated = await response.json();
```

Antwort:

```json
[
  {
    "id": 18,
    "title": "Geänderter Titel",
    "category": "Surrealer Traum",
    "shortSummary": "Neue Kurzbeschreibung.",
    "fullSummary": "Neue ausführliche Zusammenfassung.",
    "transcript": "Aktualisiertes Transkript.",
    "keyTopics": ["Thema 1", "Thema 2"],
    "people": ["Person 1"],
    "places": ["Ort 1"],
    "mood": ["Verwirrung"],
    "actionItems": [],
    "reflectionNotes": ["Neue Reflexion."],
    "imageStatus": "ready",
    "imageUrl": "https://<deine-public-bucket-domain>/dream-18.png"
  }
]
```

---

### 8.4 Note löschen

```http
DELETE /webhook/notes?id=<NOTE_ID>
```

Beispiel mit `fetch`:

```js
await fetch(`${N8N_API_URL}/webhook/notes?id=${id}`, {
  method: "DELETE",
  headers: {
    "x-api-key": import.meta.env.VITE_N8N_API_KEY,
  },
});
```

Die Response kommt direkt aus n8n/Data Table. Für das Frontend reicht in der Regel:

```js
if (!response.ok) {
  throw new Error("Note could not be deleted");
}
```

Danach am besten die Liste erneut mit `GET /webhook/notes` laden.

---

## 9. Empfohlene Frontend-Umgebungsvariablen

Beispiel für `.env`:

```env
VITE_N8N_API_URL=https://<deine-n8n-domain>
VITE_N8N_API_KEY=<DEIN_API_KEY>
```

Beispiel-Helper:

```js
const API_BASE = `${import.meta.env.VITE_N8N_API_URL}/webhook/notes`;

export async function getNotes() {
  const response = await fetch(API_BASE, {
    headers: {
      "x-api-key": import.meta.env.VITE_N8N_API_KEY,
    },
  });

  if (!response.ok) {
    throw new Error("Could not fetch notes");
  }

  return response.json();
}

export async function createNote(note) {
  const response = await fetch(API_BASE, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": import.meta.env.VITE_N8N_API_KEY,
    },
    body: JSON.stringify(note),
  });

  if (!response.ok) {
    throw new Error("Could not create note");
  }

  return response.json();
}

export async function updateNote(id, note) {
  const response = await fetch(`${API_BASE}?id=${id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": import.meta.env.VITE_N8N_API_KEY,
    },
    body: JSON.stringify(note),
  });

  if (!response.ok) {
    throw new Error("Could not update note");
  }

  return response.json();
}

export async function deleteNote(id) {
  const response = await fetch(`${API_BASE}?id=${id}`, {
    method: "DELETE",
    headers: {
      "x-api-key": import.meta.env.VITE_N8N_API_KEY,
    },
  });

  if (!response.ok) {
    throw new Error("Could not delete note");
  }
}
```

---

## 10. Quick Start Checkliste

1. `MainWorkflow.json` und `APIs.json` in n8n importieren.
2. Data Table `voice_notes_table` mit allen Spalten erstellen.
3. Alle Data-Table-Nodes auf diese Tabelle setzen.
4. Credentials anlegen:
   - Twilio API
   - Twilio Media Basic Auth
   - AssemblyAI
   - Google Gemini
   - S3/R2
   - Header Auth für Frontend
5. Twilio-Nummern in allen WhatsApp-Nodes anpassen.
6. Public Image URL im Node `Upsert row(s)` anpassen.
7. Beide Workflows aktivieren.
8. Twilio Incoming Webhook auf `/webhook/whatsapp` setzen.
9. WhatsApp-Audio senden und prüfen, ob eine neue Zeile in der Data Table entsteht.
10. Frontend mit `/webhook/notes` verbinden.

---

## 11. Häufige Fehler

### `401 Unauthorized` beim Frontend

Der Header fehlt oder ist falsch:

```http
x-api-key: <DEIN_API_KEY>
```

### WhatsApp-Audio wird nicht heruntergeladen

Prüfe den `HTTP Request`-Node. Für Twilio-Media-URLs brauchst du HTTP Basic Auth mit:

```text
Username: Twilio Account SID
Password: Twilio Auth Token
```

### Workflow findet die Tabelle nicht

Nach dem Import ist die alte Data-Table-ID ungültig. Öffne alle Data-Table-Nodes und wähle deine lokale `voice_notes_table` neu aus.

### Gemini-Node gibt kein valides JSON zurück

Der Workflow erwartet valides JSON. Prüfe den Node `Message a model` und teste den Output. Der nachfolgende Code-Node entfernt zwar Markdown-Codeblöcke, aber der Inhalt muss am Ende parsebares JSON sein.

### Bilder werden generiert, aber nicht angezeigt

Prüfe:

```text
S3/R2 Credential
Bucket-Name
öffentliche Bucket-URL im Node Upsert row(s)
Dateiname dream-<id>.png
```

---

## 12. Sicherheit

Keine echten Tokens, Telefonnummern oder Secrets in das Repository committen.

Nutze Platzhalter in den Workflow-Dateien oder dokumentiere die benötigten Werte über `.env.example`. Der Frontend-API-Key sollte regelmäßig geändert werden, wenn er versehentlich veröffentlicht wurde.
