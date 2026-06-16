# Dream Notes n8n Workflows

Dieses Repository enthält die n8n-Workflows für die Dream-Notes-App. Die eigentliche Weboberfläche liegt in einem separaten Frontend-Repository:

```text
https://github.com/JakobFischer2574/dream-notes
```

Dieses Repo ist deshalb vor allem dafür gedacht, die Backend-Automatisierung in n8n schnell einzurichten:

```text
MainWorkflow.json   # WhatsApp-Sprachnachricht → Transkript → KI-Analyse → Data Table → Bild
APIs.json           # API-Schicht für das vorhandene Frontend
```

Der Fokus dieser README liegt nicht auf der internen Node-Logik und auch nicht auf einer vollständigen API-Dokumentation. Ziel ist, die beiden Workflows möglichst schnell mit den richtigen Credentials zum Laufen zu bringen und sie mit dem bestehenden Frontend zu verbinden.

\---

## 1\. Gesamtarchitektur

```text
WhatsApp-Sprachnachricht
        ↓
MainWorkflow.json
        ↓
n8n Data Table: voice\_notes\_table
        ↓
APIs.json
        ↓
Frontend: github.com/JakobFischer2574/dream-notes
```

Der Main Workflow verarbeitet neue WhatsApp-Sprachnachrichten. Er lädt das Audio herunter, erstellt ein Transkript, lässt den Traum durch Gemini strukturieren, speichert die Notiz in einer n8n Data Table und generiert optional ein Bild.

Der API Workflow stellt die gespeicherten Dream Notes für das Frontend bereit. Das Frontend muss daher nicht direkt mit der Data Table sprechen, sondern nur mit dem API Workflow in n8n.

\---

## 2\. Voraussetzungen

Du brauchst folgende Dienste und Zugänge:

|Dienst|Zweck|
|-|-|
|n8n|Workflow-Ausführung und Data Table|
|Twilio WhatsApp|Empfang und Versand von WhatsApp-Nachrichten|
|AssemblyAI|Transkription der Audiodateien|
|Google Gemini API|Strukturierte Traum-Analyse und Bildgenerierung|
|S3-kompatibler Speicher, z. B. Cloudflare R2|Speicherung der generierten Bilder|
|Frontend-Repo|Weboberfläche für die Dream Notes|

Zusätzlich sollte deine n8n-Instanz von außen erreichbar sein, idealerweise über HTTPS. Das ist wichtig, damit Twilio und das Frontend die n8n-Webhooks erreichen können.

\---

## 3\. Workflows in n8n importieren

1. n8n öffnen.
2. Neuen Workflow erstellen.
3. `Import from File` oder `Import from JSON` auswählen.
4. `MainWorkflow.json` importieren.
5. `APIs.json` importieren.
6. Beide Workflows zunächst deaktiviert lassen.
7. Danach zuerst Data Table und Credentials einrichten.
8. Erst anschließend beide Workflows aktivieren.

Nach dem Import sind die Credential-Verweise aus der ursprünglichen n8n-Instanz noch vorhanden, aber die echten Secrets werden nicht mit exportiert. Du musst die Credentials daher in deiner eigenen n8n-Instanz neu anlegen oder neu zuweisen.

\---

## 4\. Data Table anlegen

Lege in n8n eine Data Table mit dem Namen an:

```text
voice\_notes\_table
```

Die Tabelle sollte diese Spalten enthalten:

|Spalte|Typ|
|-|-|
|title|string|
|category|string|
|shortSummary|string|
|fullSummary|string|
|transcript|string|
|keyTopics|string|
|people|string|
|places|string|
|mood|string|
|actionItems|string|
|reflectionNotes|string|
|imageStatus|string|
|imageUrl|string|

Die Felder `keyTopics`, `people`, `places`, `mood`, `actionItems` und `reflectionNotes` sind fachlich Arrays. Da n8n Data Tables hier mit String-Feldern arbeiten, werden diese Werte als JSON-String gespeichert und beim Abruf durch den API Workflow wieder in Arrays umgewandelt.

Nach dem Import musst du in allen Data-Table-Nodes deine neu angelegte Tabelle auswählen. Die exportierte interne Table-ID aus der ursprünglichen n8n-Instanz funktioniert in deiner Instanz nicht automatisch.

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

\---

## 5\. Credentials einrichten

### 5.1 Twilio API

Wird für WhatsApp-Nachrichten verwendet.

In n8n anlegen:

```text
Credential type: Twilio API
Name: Twilio account
```

Danach dieses Credential in allen Twilio-Nodes im Main Workflow auswählen.

Betroffene Nodes:

```text
To WA: Sprachaufnahme erhalten
To WA: Media Fehler
To WA: Transkription fertig
Fehler: Gemini
To WA: Dein Traum wurde erstellt.
To WA: Bildgenerierung wird gestartet.1
```

Passe in diesen Nodes außerdem die Telefonnummern an:

```text
from: deine Twilio-WhatsApp-Nummer
to: deine Ziel-WhatsApp-Nummer
```

\---

### 5.2 HTTP Basic Auth für Twilio-Media-Download

Der Node `HTTP Request` lädt die Audiodatei herunter, die Twilio als Media-URL bereitstellt.

In n8n anlegen:

```text
Credential type: HTTP Basic Auth
Name: Twilio Media Basic Auth
Username: TWILIO\_ACCOUNT\_SID
Password: TWILIO\_AUTH\_TOKEN
```

Dieses Credential anschließend im Node `HTTP Request` auswählen.

\---

### 5.3 AssemblyAI

Wird für die Transkription der Audiodateien verwendet.

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

Falls der AssemblyAI-Node in deiner n8n-Instanz fehlt, muss das passende Community Node Package installiert werden.

\---

### 5.4 Google Gemini

Wird für die strukturierte Traum-Analyse und für die Bildgenerierung verwendet.

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

Im Workflow sind diese Modelle vorgesehen:

```text
models/gemini-2.5-flash
models/gemini-2.5-flash-image
```

Falls diese Modelle in deiner Umgebung nicht verfügbar sind, wähle im jeweiligen Node ein verfügbares Gemini-Modell aus.

\---

### 5.5 S3 / Cloudflare R2

Wird verwendet, um die generierten Bilder hochzuladen.

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

Passe außerdem im Node `Upsert row(s)` die öffentliche Bild-URL an. Dort muss deine öffentlich erreichbare Bucket-URL stehen, damit das Frontend die generierten Bilder anzeigen kann.

\---

### 5.6 Header Auth für das Frontend

Der API Workflow ist durch Header Auth geschützt.

In n8n anlegen:

```text
Credential type: Header Auth
Name: Header Auth account
Header Name: x-api-key
Header Value: ein langes geheimes Token
```

Diesen API-Key trägst du später im Frontend als Umgebungsvariable ein. Er sollte nicht direkt in den Quellcode geschrieben werden.

\---

## 6\. Main Workflow: WhatsApp verbinden

Der Einstiegspunkt für WhatsApp ist der Webhook im Main Workflow:

```text
POST /webhook/whatsapp
```

Die vollständige Production-URL sieht typischerweise so aus:

```text
https://<deine-n8n-domain>/webhook/whatsapp
```

Diese URL muss in Twilio als Incoming-Message-Webhook für WhatsApp eingetragen werden.

Testablauf:

1. Main Workflow aktivieren.
2. Twilio WhatsApp Sandbox oder produktive Twilio-WhatsApp-Nummer konfigurieren.
3. Eine WhatsApp-Sprachnachricht an die Twilio-Nummer senden.
4. Prüfen, ob n8n eine neue Zeile in `voice\_notes\_table` erstellt.
5. Prüfen, ob nach kurzer Zeit auch `imageStatus` und `imageUrl` gesetzt werden.
6. Danach im Frontend prüfen, ob die neue Dream Note angezeigt wird.

Hinweis: Im exportierten Main Workflow gibt es zusätzlich einen `Twilio Trigger`-Node. Für den hier beschriebenen Ablauf ist aber der normale Webhook-Node `POST /webhook/whatsapp` der relevante Einstiegspunkt.

\---

## 7\. API Workflow mit dem Frontend verbinden

Das Frontend liegt hier:

```text
https://github.com/JakobFischer2574/dream-notes
```

Der API Workflow aus `APIs.json` stellt die Dream Notes für dieses Frontend bereit. Du musst im Frontend im Wesentlichen nur die n8n-Basis-URL und den API-Key eintragen.

Beispiel für die benötigten Frontend-Umgebungsvariablen:

```env
VITE\_N8N\_API\_URL=https://<deine-n8n-domain>
VITE\_N8N\_API\_KEY=<DEIN\_API\_KEY>
```

Das Frontend verwendet diese Werte, um den API Workflow in n8n zu erreichen. Die API-Logik ist bereits im Frontend-Repo umgesetzt, deshalb wird sie hier nicht noch einmal im Detail dokumentiert.

Wichtig ist nur:

|Einstellung|Bedeutung|
|-|-|
|`VITE\_N8N\_API\_URL`|Domain deiner n8n-Instanz|
|`VITE\_N8N\_API\_KEY`|Wert aus dem Header-Auth-Credential des API Workflows|

Achte darauf, dass `VITE\_N8N\_API\_URL` nur die Basis deiner n8n-Instanz enthält. Falls das Frontend den Pfad selbst ergänzt, sollte dort also nicht zusätzlich `/webhook/notes` eingetragen werden.

\---

## 8\. Frontend starten

Frontend klonen:

```bash
git clone https://github.com/JakobFischer2574/dream-notes.git
cd dream-notes
npm install
```

Dann eine `.env`-Datei im Frontend anlegen und die n8n-Werte eintragen:

```env
VITE\_N8N\_API\_URL=https://<deine-n8n-domain>
VITE\_N8N\_API\_KEY=<DEIN\_API\_KEY>
```

Frontend lokal starten:

```bash
npm run dev
```

Wenn das Frontend produktiv deployed wird, müssen dieselben Umgebungsvariablen auch beim jeweiligen Hoster gesetzt werden.

\---

## 9\. Was das Frontend vom API Workflow bekommt

Das Frontend erhält Dream Notes mit den wichtigsten Feldern für Anzeige, Bearbeitung und Bilddarstellung:

|Feld|Bedeutung|
|-|-|
|`id`|interne ID der Note|
|`title`|kurzer Titel|
|`category`|Kategorie des Traums|
|`shortSummary`|kurze Zusammenfassung|
|`fullSummary`|ausführlichere Zusammenfassung|
|`transcript`|ursprüngliches Transkript|
|`keyTopics`|Themen als Array|
|`people`|Personen als Array|
|`places`|Orte als Array|
|`mood`|Stimmungen als Array|
|`actionItems`|mögliche Aufgaben als Array|
|`reflectionNotes`|Reflexionsnotizen als Array|
|`imageStatus`|Status der Bildgenerierung, z. B. `pending` oder `ready`|
|`imageUrl`|öffentliche URL zum generierten Bild|

Für die normale Nutzung reicht es, wenn das Frontend diese Daten lesen, anzeigen, bearbeiten und löschen kann. Die genaue Request-Logik ist im Frontend-Repo enthalten.

\---

## 10\. Wichtig: Array-Felder sauber speichern

Für das Frontend sollten diese Felder als Arrays zurückkommen:

```text
keyTopics
people
places
mood
actionItems
reflectionNotes
```

Deshalb sollten sie in der Data Table als JSON-String gespeichert werden, nicht nur als kommaseparierter Text.

Empfohlene n8n-Expression für diese Felder:

```js
{{ JSON.stringify($json.body.keyTopics) }}
```

statt:

```js
{{ $json.body.keyTopics.join(', ') }}
```

Das betrifft vor allem die Nodes `Insert row` und `Update row(s)` im API Workflow. Wenn dort nur `.join(', ')` verwendet wird, sehen die Werte zwar lesbar aus, lassen sich später aber nicht zuverlässig wieder als Arrays parsen.

\---

## 11\. Quick Start Checkliste

1. `MainWorkflow.json` und `APIs.json` in n8n importieren.
2. Data Table `voice\_notes\_table` mit allen Spalten erstellen.
3. Alle Data-Table-Nodes auf diese Tabelle setzen.
4. Credentials anlegen und zuweisen:

   * Twilio API
   * Twilio Media Basic Auth
   * AssemblyAI
   * Google Gemini
   * S3/R2
   * Header Auth für das Frontend
5. Twilio-Nummern in den WhatsApp-Nodes anpassen.
6. Public Image URL im Node `Upsert row(s)` anpassen.
7. Main Workflow aktivieren.
8. Twilio Incoming Webhook auf `/webhook/whatsapp` setzen.
9. API Workflow aktivieren.
10. Frontend aus `https://github.com/JakobFischer2574/dream-notes` klonen.
11. Im Frontend `.env` mit n8n-Domain und API-Key setzen.
12. WhatsApp-Audio senden und prüfen, ob eine neue Note in der Data Table entsteht.
13. Frontend starten und prüfen, ob die Note angezeigt wird.

\---

## 12\. Häufige Fehler

### Frontend bekommt `401 Unauthorized`

Der API-Key fehlt oder stimmt nicht mit dem Header-Auth-Credential im API Workflow überein. Prüfe `VITE\_N8N\_API\_KEY` im Frontend.

\---

### Frontend erreicht n8n nicht

Prüfe `VITE\_N8N\_API\_URL`. Der Wert sollte auf deine n8n-Instanz zeigen. Wenn im Frontend-Code der Webhook-Pfad bereits ergänzt wird, darfst du ihn nicht zusätzlich in der Umgebungsvariable eintragen.

\---

### WhatsApp-Audio wird nicht heruntergeladen

Prüfe den `HTTP Request`-Node im Main Workflow. Für Twilio-Media-URLs wird HTTP Basic Auth mit Twilio Account SID und Twilio Auth Token benötigt.

\---

### Workflow findet die Tabelle nicht

Nach dem Import ist die alte Data-Table-ID ungültig. Öffne alle Data-Table-Nodes und wähle deine lokale `voice\_notes\_table` neu aus.

\---

### Gemini-Node gibt kein valides JSON zurück

Der Workflow erwartet valides JSON. Prüfe den Output des Nodes `Message a model`. Der nachfolgende Code-Node entfernt zwar Markdown-Codeblöcke, aber der Inhalt muss am Ende parsebares JSON sein.

\---

### Bilder werden generiert, aber im Frontend nicht angezeigt

Prüfe:

```text
S3/R2 Credential
Bucket-Name
öffentliche Bucket-URL im Node Upsert row(s)
Dateiname dream-<id>.png
imageUrl in der Data Table
```

\---

## 13\. Sicherheit

Keine echten Tokens, Telefonnummern oder Secrets in das Repository committen.

Besonders schützen solltest du:

```text
Twilio Account SID / Auth Token
AssemblyAI API Key
Google Gemini API Key
S3/R2 Access Key und Secret
Frontend API Key aus Header Auth
private Telefonnummern
```

Nutze Platzhalter in den Workflow-Dateien und setze echte Werte nur direkt in n8n bzw. als Umgebungsvariablen beim Frontend-Deployment.

