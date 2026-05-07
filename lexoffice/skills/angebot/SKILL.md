---
name: angebot
description: Erstellt Angebote in Lexware Office (Lexoffice) mit korrekter Kundenanlage, Steuerlogik (§13b für Bauleistende) und Deeplink-Ausgabe. Diesen Skill IMMER verwenden, wenn der User ein Angebot, Kostenvoranschlag oder Quotation in Lexware/Lexoffice erstellen will — auch wenn er nur sagt "leg ein Angebot an", "mach mir ein Angebot für [Kunde]", "Lexware Angebot", "neuer Auftrag in Lexoffice", "Kostenvoranschlag schreiben", "Bauleistung anbieten", "Offer schreiben". Triggert auch, wenn er einen neuen Kunden + Positionen + Bauvorhaben in einem Schritt nennt. NICHT verwenden für: Rechnungen aus Dimacon-Jobs (→ dimacon:abrechnung), Angebote aus Dimacon-Planungs-URLs (→ dimacon:lexware), neue Aufträge planen (→ dimacon:planung), LV-basierte Angebote (→ lexoffice:angebot-lv).
---

# ACADEMY ABRECHNUNG

Pflichtworkflow für jedes Angebot in Lexware Office. Die Reihenfolge ist nicht optional — wenn der Kontakt nicht korrekt vorbereitet ist, lehnt die Lexoffice-API §13b-Bauleistungen ab und das Angebot scheitert.

## Reihenfolge

1. **Kunde prüfen** — gibt's den Kontakt schon?
1b. **Adresse recherchieren** — bei neuem Kontakt: WebSearch nutzen.
2. **Steuerstatus klären** — Bauleistender oder normaler Kunde? Bei Unsicherheit: nachfragen.
3. **Preise aus Lexoffice laden** — Artikelstamm abrufen, niemals den User nach Preisen fragen.
4. **Kontakt anlegen oder updaten** — mit korrektem `allowTaxFreeInvoices`-Flag.
5. **Angebot anlegen** — mit passendem `taxType`.
6. **Deeplink ausgeben** — im richtigen Format.
7. **Bei Cowork:** Live-Artefakt direkt rendern.

---

## Schritt 1 — Kunde prüfen

Bevor irgendetwas anderes passiert: schauen, ob der Kontakt schon existiert. Niemals doppelt anlegen.

```
lexoffice_list-contacts(name: "<Firmenname>", customer: true, size: 50)
```

`name` ist eine Substring-Suche. Bei Mehrtreffern dem User die Liste zeigen und fragen, welcher gemeint ist. Bei Null Treffern → Schritt 1b.

## Schritt 1b — Adresse recherchieren (bei neuem Kontakt)

Wenn der Kontakt noch nicht in Lexoffice existiert: **IMMER zuerst per WebSearch nach der Firma recherchieren**, bevor der Kontakt angelegt wird. Firmenadaten sind fast immer im Netz auffindbar — provisorische Adressen sind der Ausnahmefall, nicht die Regel. Der User erwartet, dass Claude selbstständig recherchiert, statt nach der Adresse zu fragen.

```
WebSearch(query: "<Firmenname> Adresse Telefon")
```

**Was die Recherche liefern soll:**
- Vollständiger Firmenname (inkl. Rechtsform: GmbH, AG, etc.)
- Straße + Hausnummer
- PLZ + Ort
- Telefonnummer
- E-Mail (falls auffindbar)

**Bei mehreren Standorten:** Den Standort wählen, der geografisch zur Baustelle passt — die Baustellenadresse ist ein starker Hinweis. Im Zweifel kurz den User fragen, welcher Standort gemeint ist.

**Nur wenn die Websuche wirklich gar nichts liefert** (z.B. bei Ein-Mann-Betrieben ohne Webpräsenz): Provisorisch die Baustellenadresse eintragen und in `note` vermerken, dass die echte Rechnungsadresse noch ergänzt werden muss.

## Schritt 2 — Steuerstatus klären

Bevor der Kontakt angelegt wird, muss klar sein: **Ist der Kunde Bauleistender im Sinne von §13b UStG?**

**Bauleistender = ja, wenn:**
- Bauunternehmen, Generalunternehmer, ARGE, Bauträger, Schlüsselfertigbau
- Tiefbau, Hochbau, Rohbau, Innenausbau-Firmen
- Garten- und Landschaftsbau (bei baulichen Leistungen)
- Andere Subunternehmer im Bau

**Bauleistender = nein, wenn:**
- Privatperson
- Stadt/Gemeinde/öffentliche Hand (in der Regel)
- Industrieunternehmen ohne Bauleistungs-Tätigkeit (z.B. Brauerei, Chemie)
- Hausverwaltung, Architekt, Ingenieurbüro

**Bei Unsicherheit immer fragen** — eine falsche Steuerregelung produziert falsche Belege und Korrektur-Aufwand. Lieber 5 Sekunden nachfragen als später eine Stornorechnung schreiben.

Eine kurze Rückfrage reicht: *"Ist [Kunde] ein Bauleistender (§13b)? — sonst nehme ich Standard 19% USt."*

## Schritt 3 — Preise aus Lexoffice laden (Pflicht!)

**Vor dem Erstellen des Angebots IMMER die Artikelpreise aus dem Lexoffice-Artikelstamm laden.** SSB Fidan hat alle Artikel mit Preisen in Lexoffice hinterlegt. Niemals den User nach Preisen fragen — die Preise kommen aus dem System.

```
lexoffice_list-articles(size: 250)
```

Die Artikelliste kann sehr groß sein (2000+ Zeilen). Das ist normal und kein Fehler. Wenn die Antwort zu groß für den Kontext ist, wird sie automatisch in eine Datei geschrieben — dann per Grep oder Subagent nach den relevanten Artikelnummern suchen:

**Wichtige Artikelnummern-Bereiche:**
- `10xxx` — Anfahrt/Baustelleneinrichtung (z.B. 10100 = Anfahrt bis 100km, 10025 = Anfahrt bis 25km)
- `21xxx` — Kernbohrung Stahlbeton (z.B. 21200 = ø200mm, Einheit: cm)
- `22xxx` — Kernbohrung Mauerwerk
- `30100` — Wandsäge (Einheit: m²)
- `30200` — Seilsäge (Einheit: m²)
- `30300` — Kettensäge (Einheit: cm)
- `30400` — Fugenschneider bis 7cm (Einheit: m²)
- `30500` — Fugenschneider bis 27cm (Einheit: m²)
- `99100` — Nebenarbeiten/Regiestunden (Einheit: Stunden)

**Wenn die Ergebnisse in eine Datei geschrieben werden:** Einen Subagent (Agent-Tool) beauftragen, die Datei in Chunks zu lesen und die relevanten Artikel mit articleNumber, title, unitName und netAmount zu extrahieren. Alternativ per Grep nach den Artikelnummern suchen.

**Niemals behaupten, es gäbe keine Artikel** — es gibt welche. Wenn `lexoffice_list-articles` leer zurückkommt, liegt ein temporärer Verbindungsfehler vor → nochmal versuchen.

## Schritt 4 — Kontakt anlegen (oder updaten)

```
lexoffice_create-contact({
  "version": 0,
  "roles": { "customer": {} },
  "company": {
    "name": "<Firmenname>",
    "allowTaxFreeInvoices": true|false,
    "contactPersons": [...]
  },
  "addresses": {
    "billing": [{
      "street": "...",
      "zip": "...",
      "city": "...",
      "countryCode": "DE"
    }]
  },
  "emailAddresses": { "business": [...] },
  "phoneNumbers": { "business": [...] }
})
```

**`allowTaxFreeInvoices`-Regel:**
- Bauleistender → `true`
- Privatperson, Stadt, Industrie → `false` (Default reicht)

Ohne `allowTaxFreeInvoices: true` lehnt die API später `taxType: "constructionService13b"` mit folgendem Fehler ab:

> Invalid combination of tax type constructionService13b, contact id ..., taxation in country DE

**Falls der Kontakt bereits existiert, aber `allowTaxFreeInvoices` fehlt:** Kontakt updaten via `lexoffice_update-contact` — das Flag nachträglich setzen ist möglich und unproblematisch.

**Pflichtfelder Adresse:** `name` und `countryCode` müssen vorhanden sein, sonst Error 406. Die Adresse sollte aus der WebSearch-Recherche (Schritt 1b) kommen — provisorische Adressen sind nur der letzte Ausweg.

## Schritt 5 — Angebot anlegen

```
lexoffice_create-quotation({
  "voucherDate": "<heute YYYY-MM-DDTHH:mm:ss.000+02:00>",
  "expirationDate": "<heute + 30 Tage>",
  "address": {
    "contactId": "<UUID aus Schritt 4>",
    "name": "<Firmenname>",
    "countryCode": "DE"
  },
  "lineItems": [...],
  "totalPrice": { "currency": "EUR" },
  "taxConditions": {
    "taxType": "constructionService13b" | "net" | "gross"
  },
  "shippingConditions": {
    "shippingDate": "<heute>",
    "shippingType": "service"
  },
  "title": "Angebot",
  "introduction": "...",
  "remark": "..."
})
```

**`taxType`-Wahl:**

| Kunde | taxType | LineItem-Steuersatz |
|---|---|---|
| Bauleistender §13b | `constructionService13b` | `0` |
| Standard B2B/Privat | `net` | `19` (oder `7`, `0` je nach Position) |
| Bruttoangebot (selten) | `gross` | siehe unten |

`vatfree` ist für SSB Fidan organisationsweit gesperrt — nicht verwenden.

**LineItem-Struktur:**

```json
{
  "type": "custom",
  "name": "<Kurztitel>",
  "description": "<Detailbeschreibung mit Art.-Nr.>",
  "quantity": <Zahl>,
  "unitName": "<Stück|cm|m²|Stunden|...>",
  "unitPrice": {
    "currency": "EUR",
    "netAmount": <Betrag>,
    "taxRatePercentage": <0|7|19>
  }
}
```

**Preise kommen aus dem Artikelstamm (Schritt 3)** — die `netAmount`-Werte direkt aus den Lexoffice-Artikeln übernehmen. Den User nicht nach Preisen fragen, wenn die Artikel bereits geladen sind.

**Erste LineItem als Bauvorhabens-Header:** Der erste Eintrag im Angebot ist ein `type: "text"`-Block, der das Bauvorhaben benennt:

```json
{
  "type": "text",
  "name": "BV <Bezeichnung>, <Adresse>",
  "description": "für oben genanntes Bauvorhaben unterbreiten wir Ihnen folgendes Angebot:"
}
```

**Artikelnummern verwenden:** Die Lexware-Artikelnummern (10100 Anfahrt, 21xxx Stahlbeton, 22xxx Mauerwerk, 30100 Wandsäge usw.) gehören in die `description`, damit beim späteren Abgleich mit Dimacon klar ist, welcher Artikel gemeint war. Die ø-Werte aus dem Sortiment sind: 50/80/102/125/152/200/250/300/350/400/450/500/600/700/800/1000 mm — wenn der User "ø150" sagt, ist `21152` (ø152 mm) gemeint, weil DN150-Durchbrüche in der Praxis mit ø152-Krone gebohrt werden.

## Schritt 6 — Deeplink ausgeben

Die Lexoffice-API liefert über `lexoffice_deeplink-quotation` URLs auf `app.lexware.io/permalink/...` — **diese funktionieren NICHT**. Der Link, der tatsächlich öffnet, ist:

```
https://app.lexware.de/voucher/#/<quotation-id>
```

Diesen Link selbst zusammenbauen, nicht auf den API-Deeplink verlassen.

**Für Kontakte:** Es gibt keinen funktionierenden Deeplink. Keinen Kontakt-Link generieren — nur erwähnen, dass der Kontakt angelegt wurde, ohne URL.

## Schritt 7 — Cowork: Interaktives Live-Artefakt (Pflicht in Cowork!)

Wenn die Konversation in Cowork läuft (erkennbar am Vorhandensein von `create_artifact`, `present_files` oder ähnlichen Cowork-Tools), **IMMER ein interaktives Live-Artefakt rendern** — nicht nur den Lexware-Link ausgeben. Das Artefakt ist keine optionale Zugabe, sondern ein erwarteter Bestandteil des Outputs in Cowork.

### Interaktivität — das Artefakt ist ein Editor, kein PDF

Das Artefakt muss **editierbar** sein. Der User soll Positionen direkt im Artefakt ändern können, ohne in Lexware wechseln zu müssen. Änderungen werden über `window.cowork.callMcpTool()` direkt nach Lexware geschrieben.

**Editierbare Felder (alle per `contenteditable`, KEIN `prompt()` — das ist in der Sandbox blockiert!):**
- Positionsname (fett, z.B. "Kernbohrung ø200mm Stahlbeton")
- Positionsdetail (grau, unter dem Namen, z.B. "Art.-Nr. 21200 — 20 Bohrungen × 30 cm tief")
- Menge (numerisch, formatiert sich beim Verlassen)
- Einheit (Text)
- Einzelpreis (numerisch, formatiert sich beim Verlassen)
- Gesamtpreis berechnet sich automatisch (nicht editierbar)

**Interaktive Aktionen:**
- **✕ Löschen** — erscheint links beim Hover über eine Zeile
- **＋ Position hinzufügen** — Button unter der Tabelle, erzeugt inline eine neue Zeile und fokussiert den Namen
- **Summen** — aktualisieren sich live bei jeder Änderung
- **"Neues Angebot speichern"** — erstellt ein neues Angebot in Lexware via `callMcpTool("lexoffice_create-quotation", {...})` und aktualisiert den Lexware-Link
- **Unsaved-Banner** — erscheint sobald etwas geändert wurde

**Wichtig:** `mcp_tools` im `create_artifact`-Call muss `lexoffice_create-quotation` enthalten, damit der Save-Button funktioniert.

### Design — Notion-Style (clean, hell, professionell)

- Heller Hintergrund (#ffffff), System-Font (-apple-system, Segoe UI, sans-serif)
- Textfarbe #37352f, Sekundärfarbe #9b9a97, Trennlinien #e9e9e7
- **Property-Rows** für Metadaten (Kunde, Adresse, Datum, Gültig bis, BV, Steuer) — je eine Zeile mit Label links, Wert rechts
- Farbige Tags: Orange (#fdecc8) für §13b, Blau (#d3e5ef) für BV
- **Callout-Box** (#f1f1ef) mit Einleitungstext
- **Saubere Tabelle** mit hellem Header (#f7f6f3), hover-Effekt
- **Beschreibungs-Spalte:** Name (fett) oben, Detail (grau, klein) darunter — in einer Zelle, nicht in zwei Spalten
- Editierbare Zellen: `contenteditable`, border transparent, hover zeigt #f7f6f3 + border, focus zeigt blauen Ring (#2383e2)
- **Summenblock** rechtsbündig mit Zwischensumme, USt und Gesamtbetrag
- **§13b-Hinweis** als gelbe Callout-Box (#fdecc8)
- **Buttons:** "In Lexware öffnen" (#2383e2) + "Neues Angebot speichern" (#0f7b0f, disabled wenn keine Änderungen)
- Kein dunkles Design, kein Barlow Condensed — Notion-Style ist Standard

In Nicht-Cowork-Umgebungen (z.B. Claude Desktop, Mobile App): kein Artefakt rendern — eine kompakte Markdown-Tabelle reicht.

---

## Häufige Fehler & Lösungen

| Fehler | Ursache | Lösung |
|---|---|---|
| `Name and countryCode are required address fields.` | `address` im Quotation hat nicht `name` + `countryCode` | Beides ergänzen — `contactId` allein reicht nicht |
| `Invalid combination of tax type constructionService13b...` | Kontakt hat `allowTaxFreeInvoices: false` | Kontakt updaten, Flag auf `true` setzen |
| `No vatfree invoices allowed for this organization` | Versuch mit `taxType: "vatfree"` | Stattdessen `constructionService13b` (Bauleistender) oder `net` mit 0% Steuer (Notlösung) verwenden |
| Permalink-URL öffnet nicht | API-Deeplink statt korrekter URL | Selbst zusammenbauen: `https://app.lexware.de/voucher/#/<id>` |
| `lexoffice_list-articles` liefert leere Liste | Temporärer Verbindungsfehler | Erneut versuchen — es gibt Artikel im System |
| Artikelliste zu groß für Kontext | Über 2000 Zeilen | Datei wird automatisch gespeichert — per Grep/Subagent nach Artikelnummern suchen |
| User wird nach Preisen gefragt | Artikel nicht geladen | Schritt 3 nachholen — Preise kommen aus dem Artikelstamm, nicht vom User |

## Rechenformeln (häufige Positionen)

- **Kernbohrung:** Tiefe in cm × Stückzahl = Gesamt-cm; Gesamt-cm × Preis/cm = Position
- **Wandsäge:** Länge × Tiefe = m² pro Schnitt; × Anzahl Schnitte = Gesamt-m²; × Preis/m² = Position
- **Anfahrt + BE:** Standardartikel `10100` (140 € bis 100km) bzw. `10025` (110 € bis 25km)
- **§13b-Hinweis im Remark:** "Steuerschuldnerschaft des Leistungsempfängers gem. § 13b Abs. 2 Nr. 4 UStG (Bauleistung)."

## Tools (ACADEMY MCP)

Folgende Lexoffice-Tools werden in diesem Workflow benötigt:

- `lexoffice_list-contacts` — Kunde suchen
- `lexoffice_get-contact` — bestehenden Kontakt prüfen
- `lexoffice_create-contact` — Kunde anlegen
- `lexoffice_update-contact` — Flag nachträglich setzen
- `lexoffice_list-articles` — Artikelstamm mit Preisen laden (Pflicht vor jedem Angebot!)
- `lexoffice_create-quotation` — Angebot erzeugen
- `lexoffice_get-quotation` — zur Verifikation
- `WebSearch` — Firmenadresse recherchieren (bei neuem Kontakt)

Falls der ACADEMY MCP nicht verfügbar ist, sind die gleichen Tools im Dimacon MCP MIRANUM verfügbar — die Tool-Namen sind identisch.
