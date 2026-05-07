---
name: angebot-lv
description: Erstellt Angebote in Lexware Office (Lexoffice) aus einem hochgeladenen LV (PDF). Daraus wird eine strukturierte Excel (Claude Desktop/Code) bzw. ein editierbares Live-Artefakt (Cowork), das per Knopfdruck zum Lexoffice-Angebot wird. Triggert bei "hier ist das LV", "kalkulier mir das LV", "Excel zum LV", "Angebot aus LV", oder wenn ein PDF mit "LV", "Leistungsverzeichnis" oder "Ausschreibung" hochgeladen wird. NICHT für: einfache Angebote ohne LV (→ lexoffice:angebot), Rechnungen aus Dimacon-Jobs (→ dimacon:abrechnung), Angebote aus Dimacon-Planungs-URLs (→ dimacon:lexware), Aufträge planen (→ dimacon:planung).
---

# ACADEMY ANGEBOT

Pflichtworkflow für jedes Angebot in Lexware Office. Der Skill verarbeitet zwei Eingangsformen und produziert je nach Umgebung unterschiedliche Outputs.

## Eingangsformen erkennen

| Was der User liefert | Pfad |
|---|---|
| Positionen im Chat ("5x Kernbohrung ø150…") | **Pfad A** — direkt zum Angebot |
| PDF/Word-Datei mit LV oder Ausschreibung | **Pfad B** — erst Excel/Artefakt, dann Angebot |
| Eine zuvor von Claude erzeugte Excel, jetzt bearbeitet zurück | **Pfad B, ab Schritt B5** |

Bei gemischten Eingaben (PDF + ergänzende Worte) immer den PDF-Pfad nehmen — die Datei ist die Quelle der Wahrheit.

## Umgebungen erkennen

| Signal | Umgebung | Output bei Pfad B |
|---|---|---|
| `visualize:show_widget` oder `create_artifact` verfügbar | **Cowork / Web-Chat mit Artefakt-Support** | Live-Artefakt (editierbar, Speichern-Button → Lexoffice) |
| `bash_tool` + `present_files` verfügbar, kein Artefakt-Tool | **Claude Desktop / Claude Code** | Excel-Datei zum Download/Bearbeiten |
| Nur Chat, keine Tools | Mobile App | Markdown-Tabelle (Notlösung) |

In Zweifelsfällen: Excel ist immer ein sinnvoller Output, weil sie auch in Cowork heruntergeladen werden kann.

---

## Reihenfolge — Pfad A: Direkt-Angebot

1. **Kunde prüfen** — Schritt 1
2. **Adresse recherchieren** (bei neuem Kontakt) — Schritt 1b
3. **Steuerstatus klären** — Schritt 2
4. **Preise aus Lexoffice laden** — Schritt 3
5. **Kontakt anlegen oder updaten** — Schritt 4
6. **Angebot anlegen** — Schritt 5
7. **Deeplink ausgeben** — Schritt 6
8. **Bei Cowork:** Live-Artefakt — Schritt 7

## Reihenfolge — Pfad B: LV-PDF → Excel/Artefakt → Angebot

- **B1.** PDF lesen, Positionen extrahieren
- **B2.** Lexoffice-Artikelstamm laden (= Schritt 3 vorgezogen)
- **B3.** Mapping LV-Position ↔ Lex-Art-Nr automatisch erzeugen
- **B4.** **Output:** Excel-Datei (Desktop/Code) ODER Live-Artefakt (Cowork)
- **B5.** User bearbeitet/bestätigt → bei Excel-Pfad: bearbeitete Excel zurück → einlesen
- **B6.** Steuerstatus klären — Schritt 2
- **B7.** Kunde prüfen + ggf. anlegen — Schritt 1, 1b, 4
- **B8.** Angebot anlegen — Schritt 5
- **B9.** Deeplink — Schritt 6

Details zu Schritten 1–7 stehen unten. Die Pfad-B-spezifischen Schritte folgen direkt darunter.

---

## Schritt 1 — Kunde prüfen

```
lexoffice_list-contacts(name: "<Firmenname>", customer: true, size: 50)
```

Substring-Suche. Bei Mehrtreffern dem User die Liste zeigen und fragen. Bei Null Treffern → Schritt 1b.

## Schritt 1b — Adresse recherchieren (neuer Kontakt)

**IMMER zuerst per WebSearch nach der Firma recherchieren**, bevor der Kontakt angelegt wird. Firmenadressen sind fast immer im Netz auffindbar.

```
WebSearch(query: "<Firmenname> Adresse Telefon")
```

Liefern soll: Vollständiger Firmenname (mit Rechtsform), Straße + Hausnr., PLZ + Ort, Telefon, E-Mail.

Bei mehreren Standorten den geografisch zur Baustelle passenden wählen. Im Zweifel User fragen. Nur wenn die Suche wirklich nichts liefert: Provisorisch Baustellenadresse + `note`-Vermerk.

## Schritt 2 — Steuerstatus klären

**Bauleistender im Sinne von §13b UStG?**

Ja: Bauunternehmen, GU, ARGE, Bauträger, Schlüsselfertigbau, Tiefbau, Hochbau, Rohbau, Innenausbau, GaLaBau (bei baulichen Leistungen), Bau-Subunternehmer.

Nein: Privatperson, Stadt/Gemeinde/öffentliche Hand, Industrie ohne Bautätigkeit, Hausverwaltung, Architekt, Ingenieurbüro.

**Bei Unsicherheit immer fragen:** *"Ist [Kunde] ein Bauleistender (§13b)? — sonst nehme ich Standard 19% USt."*

## Schritt 3 — Preise aus Lexoffice laden (Pflicht!)

```
lexoffice_list-articles(size: 250)
```

Niemals den User nach Preisen fragen — die Preise kommen aus dem System. Bei sehr großen Antworten: per Subagent oder Grep filtern.

**Artikelnummern-Bereiche:**
- `10xxx` — Anfahrt/BE
- `21xxx` — Kernbohrung Stahlbeton (cm)
- `22xxx` — Kernbohrung Mauerwerk (cm)
- `30100` — Wandsäge (m²), `30200` — Seilsäge (m²), `30300` — Kettensäge (cm)
- `30400/30500` — Fugenschneider (m²)
- `40xxx-42xxx` — Aufwand/Befestigung/Geräte
- `99100` — Nebenarbeiten (Std)

**Kernbohrungs-ø-Werte:** 50/80/102/125/152/200/250/300/350/400/450/500/600/700/800/1000 mm. "ø150" → `21152` (DN150-Durchbruch wird mit ø152-Krone gebohrt).

## Schritt 4 — Kontakt anlegen oder updaten

```json
{
  "version": 0,
  "roles": { "customer": {} },
  "company": {
    "name": "<Firmenname>",
    "allowTaxFreeInvoices": true,
    "contactPersons": [...]
  },
  "addresses": {
    "billing": [{ "street": "...", "zip": "...", "city": "...", "countryCode": "DE" }]
  },
  "emailAddresses": { "business": [...] },
  "phoneNumbers": { "business": [...] }
}
```

`allowTaxFreeInvoices: true` für Bauleistende — sonst lehnt die API `taxType: "constructionService13b"` ab. Falls der Kontakt schon existiert, aber das Flag fehlt: per `lexoffice_update-contact` nachsetzen. Pflichtfelder Adresse: `name` + `countryCode`.

## Schritt 5 — Angebot anlegen

```json
{
  "voucherDate": "<heute YYYY-MM-DDTHH:mm:ss.000+02:00>",
  "expirationDate": "<heute + 30 Tage>",
  "address": { "contactId": "...", "name": "...", "countryCode": "DE" },
  "lineItems": [...],
  "totalPrice": { "currency": "EUR" },
  "taxConditions": { "taxType": "constructionService13b" | "net" },
  "shippingConditions": { "shippingDate": "<heute>", "shippingType": "service" },
  "title": "Angebot",
  "introduction": "...",
  "remark": "..."
}
```

| Kunde | taxType | LineItem-Steuersatz |
|---|---|---|
| Bauleistender §13b | `constructionService13b` | `0` |
| Standard B2B/Privat | `net` | `19` (oder `7`/`0`) |

`vatfree` ist organisationsweit gesperrt — nicht verwenden.

**Erste LineItem als BV-Header:**
```json
{ "type": "text", "name": "BV <Bezeichnung>, <Adresse>", "description": "für oben genanntes Bauvorhaben unterbreiten wir Ihnen folgendes Angebot:" }
```

**Datenpositionen:**
```json
{
  "type": "custom",
  "name": "<Kurztitel>",
  "description": "<Detail mit Art.-Nr.>",
  "quantity": <Zahl>,
  "unitName": "<Stück|cm|m²|Std|...>",
  "unitPrice": { "currency": "EUR", "netAmount": <Betrag>, "taxRatePercentage": <0|7|19> }
}
```

**Gruppentitel zwischen Positionen:** Weitere `type: "text"`-Items als Untergruppen-Header (z.B. "Kernbohrungen Stahlbeton", "Wandsägen", "Aufwände") — folgt der Excel-Struktur aus Pfad B.

**§13b-Hinweis im Remark:** `"Steuerschuldnerschaft des Leistungsempfängers gem. § 13b Abs. 2 Nr. 4 UStG (Bauleistung)."`

## Schritt 6 — Deeplink ausgeben

API-Permalinks (`app.lexware.io/permalink/...`) **funktionieren nicht**. Korrekt:

```
https://app.lexware.de/voucher/#/<quotation-id>
```

Diesen Link selbst zusammenbauen. Für Kontakte gibt es keinen funktionierenden Deeplink — nicht generieren.

## Schritt 7 — Cowork: Live-Artefakt

In Cowork zusätzlich ein editierbares Artefakt rendern (Notion-Style, hell, #2383e2 Blau, #fdecc8 Orange für §13b, #0f7b0f Grün für Save-Button). Editierbare Felder via `contenteditable` (KEIN `prompt()` — Sandbox blockt das). Save-Button ruft `window.cowork.callMcpTool("lexoffice_create-quotation", {...})`. Im `create_artifact`-Call muss `mcp_tools` das Tool enthalten.

Detail-Spec siehe `references/cowork-artefakt.md`.

---

## Pfad B: LV-PDF → Excel/Artefakt → Angebot

### B1 — PDF lesen, Positionen extrahieren

PDF mit `pdf-reading`-Skill öffnen. Aus jeder LV-Position folgende Felder extrahieren:

| Feld | Pflicht | Beispiel |
|---|---|---|
| LV-Pos | ja | `2.1.112` |
| Bezeichnung | ja | `Kernbohrung Stahlbeton D=143-152 mm` |
| Material | nein | `Stahlbeton`, `Mauerwerk`, leer |
| Einheit | ja | `cm`, `m²`, `Stk`, `Std` |
| Menge (LV) | ja | `811.5` |
| EP (Bieterpreis) | nein | leer im LV — kommt von uns |

Mehrseitige LVs: Reihenfolge der Positionen beibehalten. Gruppenüberschriften (z.B. "Titel 2.1 Kernbohrungen Stahlbeton") als Excel-Gruppentitel übernehmen — nicht ignorieren.

### B2 — Artikelstamm laden

= Schritt 3.

### B3 — Mapping LV-Position ↔ Lex-Art-Nr

Für jede LV-Position automatisch passenden Lex-Artikel finden anhand:
- Material (Stahlbeton vs. Mauerwerk)
- Einheit (cm vs. m² vs. Std)
- Größe/Durchmesser (z.B. "D=143-152" → ø152 → `21152`)

**Mapping-Tabelle in `references/lv-mapping.md`** für die typischen Bezeichnungen aus deutschen LVs.

Bei Positionen ohne sicheres Mapping: Die Lex-Art-Nr leer lassen und in der Excel/dem Artefakt mit gelbem Hintergrund markieren — der User entscheidet.

### B4 — Output erzeugen

**Cowork:** Live-Artefakt mit Tabelle aller Positionen. Detail siehe `references/cowork-artefakt.md`.

**Claude Desktop / Code:** Excel-Datei nach **exakt dem Format der Referenzdatei** erzeugen. Detail-Spec mit Code-Vorlage siehe `references/excel-format.md`. Über den `xlsx`-Skill erstellen, mit `present_files` zurückgeben.

### B5 — Bearbeitete Excel einlesen

Wenn der User eine bearbeitete Excel-Datei zurückschickt: Erkennen am Header (Zeile 4: `LV-Pos | Bezeichnung | Material | Einheit | EP € | Σ Anzahl | Σ Menge | Σ Wert €`) und am BV-Titel in A1.

```python
# Lesen mit data_only=True um berechnete Werte zu bekommen
from openpyxl import load_workbook
wb = load_workbook(path, data_only=True)
ws = wb.active
# A1: BV-Titel; A2: Leistungszeitraum/Untertitel
# Ab Zeile 5: Daten + Gruppentitel (Gruppentitel = nur A gefüllt)
# Ab "Zwischensumme*" in B: Aggregation, ignorieren für LineItems
```

User-Änderungen interpretieren:
- Geänderte EP-Werte → übernehmen (User kann manuell von Lex-Standard abweichen)
- Geänderte Mengen → übernehmen
- Geänderte/gelöschte Zeilen → übernehmen
- Neue Zeilen → übernehmen, auch wenn ohne LV-Pos
- Kommentare/farbig markierte Felder → User kurz fragen, was er meint

### B6–B9

= Schritte 2, 1, 1b, 4, 5, 6 wie bei Pfad A.

---

## Häufige Fehler & Lösungen

| Fehler | Ursache | Lösung |
|---|---|---|
| `Name and countryCode are required address fields.` | `address` im Quotation hat nicht `name` + `countryCode` | Beides ergänzen |
| `Invalid combination of tax type constructionService13b...` | Kontakt hat `allowTaxFreeInvoices: false` | Kontakt updaten, Flag auf `true` |
| `No vatfree invoices allowed for this organization` | `taxType: "vatfree"` | `constructionService13b` (Bau) oder `net` 0 % stattdessen |
| Permalink-URL öffnet nicht | API-Deeplink statt korrekter URL | Selbst bauen: `https://app.lexware.de/voucher/#/<id>` |
| `lexoffice_list-articles` leer | Temporärer Fehler | Erneut versuchen |
| Artikelliste zu groß | Kontextlimit | Subagent/Grep auf Datei |
| User wird nach Preisen gefragt | Artikelstamm nicht geladen | Schritt 3 nachholen |
| PDF-Position ohne Lex-Mapping | Bezeichnung weicht von Standard ab | Zelle gelb markieren, User entscheiden lassen |

## Tools (ACADEMY MCP)

- `lexoffice_list-contacts` / `_get-contact` / `_create-contact` / `_update-contact`
- `lexoffice_list-articles` (Pflicht vor jedem Angebot)
- `lexoffice_create-quotation` / `_get-quotation`
- `WebSearch` — bei neuem Kontakt
- `pdf-reading`-Skill / `pdf`-Skill — PDF-LV einlesen
- `xlsx`-Skill — Excel erzeugen und lesen
- Cowork: `create_artifact` mit `mcp_tools: ["lexoffice_create-quotation"]`

Falls der ACADEMY MCP nicht verfügbar ist: Tools sind im Dimacon MCP MIRANUM unter gleichem Namen.

## Referenzen

- `references/excel-format.md` — exakte Excel-Struktur mit Code-Vorlage (Pfad B4)
- `references/cowork-artefakt.md` — Live-Artefakt-Spec (Cowork Pfad A & B)
- `references/lv-mapping.md` — Mapping-Tabelle Standard-LV-Bezeichnungen → Lex-Art-Nr
- `scripts/build_lv_excel.py` — fertiger Generator für Pfad B (Desktop/Code)
- `scripts/read_lv_excel.py` — Re-Import bearbeiteter LV-Excel (Pfad B5)
