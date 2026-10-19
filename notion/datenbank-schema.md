# Notion-Schema

Der Workflow schreibt in zwei Notion-Ziele: eine **Datenbank** mit allen Positionen und eine **Seite** mit der KI-Empfehlung. Beide müssen mit der Notion-Integration geteilt sein (*Seite öffnen → ••• → Verbindungen → Integration hinzufügen*).

## Datenbank „Angebotsanalyse“

Eine Zeile je angebotener Leistung. Ein Angebot mit drei Leistungen ergibt drei Zeilen mit demselben Anbieter, Datum, Gesamtpreis und Hinweis.

| Spalte | Notion-Typ | Wert aus dem Workflow | Hinweis |
|---|---|---|---|
| Name | Titel | `{{ $json.anbieter }}` | Anbietername |
| Datum | Datum | `{{ $json.datum }}` | Der Prompt verlangt das Format `YYYY-MM-DD` |
| Projekt | Text | `{{ $json.projekt }}` | Bauvorhaben laut Angebot |
| Positionen | Mehrfachauswahl | `{{ $json.leistung }}` | Bezeichnung der Leistung |
| Preis pro Posiion | Zahl | `{{ $json.preis }}` | Leer, wenn das Angebot keinen Einzelpreis nennt. Schreibweise „Posiion“ wie in der Datenbank – beim Umbenennen der Spalte auch den Node anpassen |
| Gesamtpreis | Zahl | `{{ $json.gesamtpreis }}` | Leer, wenn nicht eindeutig erkennbar |
| Hinweise | Text | `{{ $json.hinweise }}` | z. B. Ausschlüsse, Lieferzeit, Gültigkeit |

Im Node *Notion Positionen anlegen* den Platzhalter `NOTION_DATENBANK_ID_ANGEBOTSANALYSE` durch die eigene Datenbank ersetzen.

## Seite „Angebotsvergleich – Empfehlung“

Eine normale Notion-Seite. Der Node *Notion Empfehlung schreiben* ersetzt bei jedem Lauf ihren Inhalt (*Update page markdown*) durch die Ausgabe des Agenten *Empfehlung erstellen*:

- **Zusammenfassung** – Unterschiede der Angebote in Leistungen, Preisen, Hinweisen und Lieferzeiten, mit Anbieternamen
- **Empfehlung** – sachliche Einschätzung, Ausschlüsse mit möglichen Zusatzkosten, Hinweis auf nicht vergleichbare Gewerke

Im Node den Platzhalter `NOTION_SEITEN_ID_EMPFEHLUNG` durch die eigene Seite ersetzen.

## JSON-Schema der Strukturierung

Der Structured Output Parser am Agenten *Angebot strukturieren* erwartet je Angebot:

```json
{
  "anbieter": "string",
  "datum": "string (YYYY-MM-DD)",
  "projekt": "string",
  "positionen": [ { "leistung": "string", "preis": "number | null" } ],
  "gesamtpreis": "number | null",
  "hinweise": "string"
}
```

Fehlende Preise werden als `null` übernommen, nicht geschätzt.
