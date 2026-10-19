# Testprotokoll

**Testumgebung:** n8n (selbst gehostet), Groq `openai/gpt-oss-120b`, Ollama `qwen3-embedding:0.6b`, Ollama `qwen2.5vl:7b` für Bilder (vorher Google Gemini), Notion
**Testdaten:** fiktive Angebote zum Bauvorhaben „Neubau Wohnanlage A“ als PDF, DOCX und PNG, bereitgestellt von der Werkskolleg GmbH für berufliche Qualifizierung (Urheberrecht beim Herausgeber, nicht im Repository)
**Vorgehen:** immer nur eine Änderung, danach derselbe Testlauf mit denselben Dateien

## Übersicht

| Nr. | Befund | Ursache | Lösung | Status |
|---|---|---|---|---|
| 1 | „Model output doesn't fit required format“ | Lokales Chat-Modell über Ollama hält das JSON-Schema nicht ein | Sprachmodelle bleiben bei Groq; Embeddings lokal | ✓ behoben |
| 2 | Retrieval-Tool hing am falschen Agenten | Vector Store war mit dem Extraktions-Agenten verbunden | An den Agenten *Empfehlung erstellen* umgehängt | ✓ behoben |
| 3 | Chat: „Bad request – please check your parameters“ | Neues Groq-Node nutzte das abgeschaltete Standardmodell `llama3-8b-8192` | Dasselbe Modell wie in den anderen Groq-Nodes gewählt | ✓ behoben |
| 4 | Von 2 Bildern kam nur 1 an | Code-Node *Bildtext aufbereiten* nutzte `$input.first()` | `$input.all().map(…)` – jedes Bild wird ein eigenes Item | ✓ behoben |
| 5 | „The service is receiving too many requests“ bei 8 Dateien | Groq Free Tier: 8.000 Tokens pro Minute | Wait-Node (60 s) vor der Empfehlung; Demo mit 5 Dateien | ◐ teilweise |
| 6 | Datum 02.04.2026 erscheint als „4. Februar 2026“ | Datumsformat nicht vorgegeben, Tag und Monat vertauscht | Prompt verlangt Format `YYYY-MM-DD` | ✓ behoben |
| 7 | Spalte *Projekt* enthält den Anbieternamen | Notion-Node schrieb `{{ $json.anbieter }}` in *Projekt* | Auf `{{ $json.projekt }}` geändert | ✓ behoben |
| 8 | Fotos als `.jpg` werden nicht verarbeitet | Switch prüfte nur auf `.png` | Bedingung `{{ /\.(png\|jpe?g)$/i.test($json.fileName) }}` mit *Boolean → is true* | ✓ behoben |
| 9 | Gemini: „Service unavailable“ | Externer Dienst überlastet (503), auch nach Modellwechsel | **Verbesserung:** Bilder lokal mit Ollama `qwen2.5vl:7b` statt Gemini | ✓ behoben |

## Details

### Befund 1 – Lokales Modell und strukturierte Ausgabe
Beim Wechsel der Chat-Modelle von Groq auf Ollama brach der Agent *Angebot strukturieren* mit „Model output doesn't fit required format“ ab. Mögliche Abhilfen (größere Kontextlänge, JSON-Modus, Auto-fixing Output Parser, größeres Modell) wurden bewusst nicht weiterverfolgt, um die Übung abzuschließen. Entscheidung: Sprachmodelle bei Groq, nur die Embeddings lokal.

### Befund 4 – Nur das erste Bild verarbeitet
Vorher:
```javascript
return [{ json: { text: $input.first().json.content.parts[0].text } }];
```
Nachher:
```javascript
return $input.all().map(item => ({
  json: { text: item.json.content.parts[0].text }
}));
```
Nachtest: *Bild auslesen* 2 Items → *Bildtext aufbereiten* 2 Items, am Merge 8 statt 7 Items.

### Befund 6 und 7 – Datum und Projekt
Nachtest mit denselben Angeboten: Schmidt Sanitärtechnik erscheint mit 2. April 2026, Bau & Trockenbau AG mit 3. April 2026; die Spalte *Projekt* zeigt „Neubau Wohnanlage A“.

### Befund 8 und 9 – Bilder
Der erste Versuch mit dem Operator *matches regex* leitete keine Bilder weiter. Mit einem JavaScript-Ausdruck und *Boolean → is true* kamen beide Bilder bei *Bild auslesen* an. Dort scheiterte der Lauf an einer Überlastung von Google Gemini (503), auch nach Wechsel auf `gemini-3.8-flash`; `gemini-2.5-flash` war bereits abgekündigt.

**Verbesserung:** Der Bildweg wurde auf ein lokales Vision-Modell umgebaut – *Bild in Base64* (Extract from File) → *Bild auslesen (lokal)* (HTTP Request an Ollama `/api/generate`, Modell `qwen2.5vl:7b`) → *Bildtext aufbereiten* (liest `response`). Damit ist der Workflow unabhängig von Gemini, und die Bilddatei verlässt den Rechner nicht.

Fehler beim Umbau: „Method not allowed“ – die URL endete auf `:11434` statt auf `/api/generate`. Nach der Korrektur lieferte der Einzeltest mit einem Foto-Angebot den vollständigen Text (Anbieter, Datum, alle Positionen mit Menge und Preis, Zwischensumme, MwSt., Gesamtpreis). Der erste Aufruf dauerte mehrere Minuten, weil Ollama das Modell erst in den Speicher lädt.

### Befund 5 – Token-Limit
Mit 8 Dateien gab *Angebot strukturieren* nur 6 statt 8 Items aus, und die Empfehlung lief ins Rate Limit. Die Wait-Node fängt das Limit vor der Empfehlung ab, nicht aber in der Strukturierung. Lösungsidee: *Loop Over Items* mit kleiner Batch-Größe und Wait im Loop, zusätzlich *Retry On Fail* an den Agenten.

## Nachtest nach den Korrekturen (3 PDF, 2 DOCX)

| Prüfung | Ergebnis |
|---|---|
| Alle Dateien verarbeitet | ✓ 5 Dateien → 5 Angebote strukturiert |
| RAG Insert schreibt alle Angebotstexte in den Vector Store | ✓ 5 Items |
| Positionen in der Notion-Datenbank angelegt | ✓ 10 Positionen |
| Datum korrekt | ✓ z. B. Schmidt Sanitärtechnik: 2. April 2026 |
| Projekt korrekt | ✓ „Neubau Wohnanlage A“ |
| Fehlende Einzelpreise bleiben leer | ✓ Sanitär & Co KG |
| Empfehlung behandelt alle Anbieter | ✓ alle 5 |
| Empfehlung nennt Ausschlüsse mit Anbieternamen | ✓ Fliesenarbeiten bei beiden Sanitärangeboten |
| Empfehlung weist auf nicht vergleichbare Gewerke hin | ✓ |
| Empfehlung nutzt das RAG-Tool | ✓ Hinweis: kein Angebot nennt Gewährleistung oder Zahlungsbedingungen |
| Chat ruft *RAG Suche Chat* auf und antwortet mit Anbieternamen | ✓ „Welcher Anbieter hat Festpreise?“ → Elektro Stark GmbH, Sanitär & Co KG |
| Chat versteht eine zweite Frage im selben Verlauf | ✓ |
| Bildweg (Gemini) | ◐ nicht getestet – Dienst überlastet, später durch Ollama ersetzt |

## Abschlusstest mit lokalem Bildweg (2 PDF, 1 DOCX, 1 Foto)

| Prüfung | Ergebnis |
|---|---|
| Alle Dateien verarbeitet | ✓ 4 Dateien → 4 Angebote strukturiert |
| Foto lokal ausgelesen (Ollama `qwen2.5vl:7b`) | ✓ Baufirma Meier mit allen 5 Positionen und Gesamtpreis |
| RAG Insert schreibt alle Angebotstexte in den Vector Store | ✓ 4 Items |
| Positionen in der Notion-Datenbank angelegt | ✓ 11 Positionen |
| Fehlende Angaben bleiben leer | ✓ Trockenbau Expert AG ohne Datum und Einzelpreis |
| Empfehlung behandelt alle Anbieter | ✓ alle 4 |
| Empfehlung erkennt veraltetes Angebot | ✓ Baufirma Meier: nur bis 25.09.2021 gültig |
| Empfehlung nennt Ausschlüsse mit Anbieternamen | ✓ Sanitärbetrieb Keller: Fliesenarbeiten nicht enthalten |
| Empfehlung weist auf unterschiedliche Gewerke hin | ✓ Einzelvergabe oder Gesamtpaket |
| Empfehlung nutzt das RAG-Tool | ✓ kein Angebot nennt Gewährleistung oder Zahlungsbedingungen |
| Chat beantwortet Fragen mit Belegen aus den Angeboten | ✓ „Welcher Anbieter arbeitet am schnellsten?“ → Trockenbau Expert AG und Trockenbau Huber, je ca. 5 Wochen, mit Zitaten |
