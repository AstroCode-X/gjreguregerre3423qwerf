# Dancing with SARS — NASA LRO + JPL Horizons + Groq KI Chat

**NASA Space Apps Challenge 2026 – Prototypweiterentwicklung auf Basis deiner ursprünglichen Dateien.**

Die dunkel-cyanfarbene Mission-Control-Oberfläche, 2D-Südpolkarte, globale 2D-Karte, der drehbare 3D-Mond, Ziele, SAR-Demo-Overlays und das lokale synthetische ML-Modell sind erhalten geblieben. Ergänzt wurden ein **echter Groq-Chat (Llama 3.3 70B)** und **Live-Abruf von NASA/JPL Horizons** sowie eine **NASA-LRO-Mondtextur** über deinen Cloudflare Worker. Vor dem Live-Einsatz musst du den Worker selbst deployen; von dieser Umgebung aus konnten wir deinen Worker oder den Groq-Account **nicht** verändern oder die NASA-/Groq-Antworten live testen.

## In 5 Minuten einrichten

### 1. Groq API-Key erstellen

Gehe auf **https://console.groq.com/keys** und erstelle einen Schlüssel. **Nicht in HTML/JavaScript kopieren und nicht in eine ZIP einfügen!** Der Key bleibt in Cloudflare als verschlüsseltes Worker-Secret.

### 2. Cloudflare Worker `apikey` aktualisieren (empfohlen: Wrangler)

**Achtung:** Der Name `apikey` ist für `https://apikey.elias256.workers.dev` vorgesehen. Kontrolliere, dass du bei Cloudflare in dem richtigen Account angemeldet bist, bevor du einen existierenden Worker überschreibst. Ein vorhandenes anderes Worker-Programm bitte vorher sichern.

In einem Terminal mit Node.js:

```bash
cd cloudflare
npm install
npx wrangler login
```

Passe jetzt im `cloudflare/wrangler.jsonc` die Variable `ALLOWED_ORIGINS` an. Trage **exakt** die später verwendeten Frontend-URLs ein, durch Kommas getrennt. Für lokale Tests sind `http://localhost:8000` und `http://127.0.0.1:8000` eingetragen. Beispiel:

```json
"ALLOWED_ORIGINS": "http://localhost:8000,http://127.0.0.1:8000,https://meine-mond-app.pages.dev"
```

Dann:

```bash
npx wrangler deploy
npx wrangler secret put GROQ_API_KEY
```

Beim zweiten Befehl den persönlichen Groq-Key **nur in der Eingabeaufforderung** eintippen. `wrangler.jsonc` enthält zwei Rate-Limit-Bindings (`CHAT_RATE_LIMITER` und `DATA_RATE_LIMITER`). Ohne `CHAT_RATE_LIMITER` verweigert der Worker aus Sicherheitsgründen Chat-Anfragen mit `503`.

Du kannst den Code auch im Cloudflare-Dashboard aus `cloudflare/worker.js` einsetzen, musst dann aber **beide Bindings**, `ALLOWED_ORIGINS` und das Secret im Dashboard konfigurieren. Wrangler ist verlässlicher, weil es die Bindings automatisch mitdeployt. Den Worker kann ChatGPT ohne Zugang zu deinem Cloudflare-Konto nicht selbst bereitstellen.

**Worker-URL (im Frontend bereits fest eingetragen):** https://apikey.elias256.workers.dev

**Test nach Deployment:** Öffne https://apikey.elias256.workers.dev/health. Dort sollten `groq_configured: true` und `chat_rate_limit_configured: true` stehen. Der API-Key selbst darf dort **niemals** zu sehen sein.

### 3. Dashboard starten

Das Frontend liegt ebenfalls fertig kopiert in `web/` (ideal für **Cloudflare Pages**). Lade **den Inhalt von `web/`** als statische Website hoch. Füge deren endgültige Origin zu `ALLOWED_ORIGINS` hinzu und deploye den Worker erneut.

Alternativ lokal:

```bash
python -m http.server 8000
```

Danach **http://localhost:8000** öffnen. Direktes Öffnen per `file://` ist für Live-API-Aufrufe standardmäßig **nicht freigegeben** – so wird der öffentliche Key-Proxy nicht für beliebige lokale Dateien aktiviert. Die 2D/3D-Fallback-Karten können auch offline gezeigt werden.

### 4. Echte Daten abrufen und mit der KI sprechen

- Beim Start wird für den aktuellen Standort automatisch `GET /ephemeris` versucht (Sonne und Erde, 1-Stunden-Takt). Der Button **„JPL LIVE LADEN“** lädt neu, **„ALLE ORTE LADEN“** lädt nacheinander alle fünf Standorte.
- JPL-Ergebnisse enthalten **Azimut und Höhe** des Sonnen- und Erdzentrums über dem **idealen geometrischen lokalen Horizont**. **Keine** lokale Geländeabschattung, kein Antennen-Modell und keine garantierte Landefähigkeit.
- Bei Fehler oder Verbindungsabbruch erscheinen `N/A` und eine Fehlermeldung, **keine als NASA ausgegebenen Fantasiewerte**. Der separate Button „DEMO-WERTE“ schaltet bewusst auf das nicht kalibrierte Modell um.
- Ein originales NASA-LRO-WAC-Mosaik wird als Textur vom Worker geladen und im Browser zum Südpolbild reprojiziert. Wenn der Worker noch nicht aktiv ist, bleibt die illustrative Offline-Ersatzkarte erhalten und wird als solche beschriftet.
- Oben auf **„KI CHAT ÖFFNEN“** klicken oder im KI-Mission-Analyst **„MIT KI SPRECHEN“**. Der Worker sendet Kontext (Ort/Datum/aktuell verfügbare Winkel und prozentuale Vergleichswerte) sowie Nachrichten an Groq. Der Key geht **nur** vom Worker zu Groq, niemals vom Browser.

## Was echte NASA-Daten sind – und was **nicht**

| Element | Status | Quelle / Einschränkung |
| --- | --- | --- |
| Mondoberflächen-Foto 2D/3D | **NASA-LRO-Mosaik, wenn online** | NASA SVS CGI Moon Kit, LROC WAC/LOLA. Auf Visualisierung optimiert; Polarregion teilweise mit LOLA-Albedo ergänzt. **Nicht** als quantitative SAR- oder Geländehöhenkarte behandeln. |
| Sonne/Erd-Positionen über dem lokalen Horizont | **NASA/JPL-Horizons-Ephemeriden nach erfolgreichem Live-Abruf** | JPL Horizons API, `QUANTITIES=4`, topozentrisch auf dem Mond, 1 Stunde. Präzise Ephemeriden sind keine Messungen der tatsächlichen Geländeverschattung. |
| Standortkoordinaten in dieser App | **Unveränderte Koordinaten aus deinem Prototyp** | Keine von NASA bestätigten CLPS-Landestellen. Die JPL-Ausgabe beschreibt die mathematische Sicht genau **an diesen frei gewählten Koordinaten**. |
| CPR-Zonen, PSR-Eisflächen, Gefahrenpolygone | **DEMO** | Aus `original/data.json`, keine validierten räumlichen NASA-Produkte. |
| Eis-Scores, Prozent-Sonneneinstrahlung, Hänge, Temperaturen, SAR-Rückstreuung | **DEMO** | Unveränderte feste Angaben aus dem ursprünglichen Python-Skript. **Keine** wissenschaftlich abgesicherten Messungen. |
| KI-Mission-Analyst Score | **DEMO-ML** | Lokale logistische Regression mit synthetischen Trainingsdaten. Kein NASA-Modell / keine reale Lande-Wahrscheinlichkeit. |
| Groq KI-Chat | **Echtes LLM**, nach erfolgreichem Worker-Setup | Llama 3.3 70B via Groq. Antworten können Fehler enthalten; hat keine selbstständige Internetrecherche und ist keine NASA-Autorität. |

Die tatsächliche NASA-Textur und die Live-JPL-Werte **werden nicht als offline vorinstallierte Messdaten ausgegeben**. Die Browserintegration ist vorbereitet, der aktive Abruf erfordert einen funktionsfähigen Worker und Netzverbindung.

## API-Endpoints deines Workers

| Methode | URL | Zweck |
| --- | --- | --- |
| GET | `/health` | Anzeigen, ob das Secret und die Rate-Limit-Bindings eingerichtet sind (keine Schlüsselwerte) |
| GET | `/texture` | Proxy und Cache für NASA SVS `lroc_color_2k.jpg` |
| GET | `/ephemeris?site=SAR-001&start=2026-11-14T00:00&hours=24` | JPL Horizons topozentrische Sonnen-/Erdwinkel für einen Standort (24/72/168 h) |
| POST | `/chat` | Groq Chat Completion mit serverseitigem Secret |

Chat-Beispiel (vom freigeschalteten Ursprung aus):

```json
{"messages":[{"role":"user","content":"Warum ist CPR kein eindeutiger Beleg für Eis?"}],"context":{"site":"Shackleton Crater Rim","source":"NASA/JPL live"}}
```

Für einen lokalen API-Test ohne Browser kannst du mit `curl` einen POST an `https://apikey.elias256.workers.dev/chat` schicken, falls das Chat-Binding und GROQ_API_KEY konfiguriert sind.

**Sicherheit:** Der Worker akzeptiert nur freigegebene Browser-Origins, begrenzt die Anfragegrößen, Nachrichtenanzahl, Ausgabelänge und Anfragerate. CORS ist **keine Authentifizierung**: ein öffentlicher Worker kann prinzipiell von Nichtbrowser-Clients direkt angesprochen werden. Bei öffentlicher Veröffentlichung zusätzlich Cloudflare-WAF/Turnstile oder eine Benutzeranmeldung einplanen, Groq-Ausgaben überwachen und Nutzungslimits konfigurieren. Chattexte und der angezeigte Dashboard-Kontext werden an Groq übertragen. Keine Passwörter, personenbezogenen oder geheimen Daten in den Chat eingeben.

## Projektstruktur

```
web/                       → statische fertige Website für Cloudflare Pages
index.html, assets/        → identische lokale Frontend-Hauptversion
cloudflare/worker.js       → Cloudflare Worker Modulskript; kein API-Key im Code
cloudflare/wrangler.jsonc  → Deployment, CORS und Rate-Limit-Bindings
cloudflare/package.json   → Wrangler-CLI-Abhängigkeit
original/                 → alle vier von dir zuerst gelieferten Dateien
assets/ml-model.js        → synthetisches lokales ML-Modell
assets/astro.js           → explizit bezeichnete Demo-Berechnung + JPL-Auswertung
tools/fetch_horizons.py   → optionaler Python-JPL-Exporter/JSON-Import
README.md                 → diese Anleitung
```

## Tests

Diese Tests liefen **ohne echten Groq-Key und ohne Live-Zugriff auf JPL**; es wurden absichtlich reproduzierbare Mock-Antworten verwendet:

```bash
node --test tests/worker.test.mjs
node --test tests/astro.test.js
python -m unittest discover -s tests -p 'test_horizons.py'
python tests/browser_smoke.py
```

Der Browser-Smoke-Test nutzt simulierte NASA/Groq-Netzantworten und fängt in dieser Sandbox blockierte Browsernavigation über ein Inline-Testdokument ab. **End-to-End mit NASA/JPL und Groq musst du nach dem Deployment selbst verifizieren.**

## Wissenschaftliche Quellen

- [NASA SVS CGI Moon Kit, 2025 LRO-WAC Mosaik (SVS 4720)](https://svs.gsfc.nasa.gov/4720)
- [NASA/JPL Horizons REST API](https://ssd-api.jpl.nasa.gov/doc/horizons.html)
- [JPL Horizons Manual – Mondoberflächen-Koordinaten](https://ssd.jpl.nasa.gov/horizons/manual.html)
- [Groq Chat Completion API](https://console.groq.com/docs/api-reference)
- [Groq Modell llama-3.3-70b-versatile](https://console.groq.com/docs/model/llama-3.3-70b-versatile)
- [Cloudflare Worker Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Cloudflare Worker Rate-Limit-Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/)

Eine Einreichung benötigt darüber hinaus überprüfte Landestellen, dokumentierte Hangneigungsdaten (z. B. hochauflösendes LRO/LOLA DEM) und Geländeabschattung. **CPR > 1 beweist kein Wassereis.**
