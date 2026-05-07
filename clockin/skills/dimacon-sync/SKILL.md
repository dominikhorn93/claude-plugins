---
name: dimacon-sync
description: >
  Synchronisiert Aufträge aus Dimacon nach Clockin als Projekte. Verwende diesen Skill immer wenn
  der Benutzer Dimacon-Aufträge nach Clockin übertragen, Clockin-Projekte aus Dimacon erstellen,
  die Tagesplanung synchronisieren, oder Aufträge in Clockin anlegen möchte. Auch bei Formulierungen
  wie "Clockin sync", "Aufträge nach Clockin", "Clockin aktualisieren", "Projekte für heute anlegen",
  "sync die Aufträge", "Clockin CSV", "Tagesplanung übertragen" oder "was steht heute an und trag
  es in Clockin ein" soll dieser Skill verwendet werden. Der Skill holt alle Termine für ein Datum
  aus Dimacon, prüft ob die Projekte in Clockin bereits existieren, und erstellt oder aktualisiert
  sie mit korrekten Mitarbeiter-Zuweisungen.
compatibility: "dimacon, clockin"
---

# Dimacon → Clockin Projekt-Synchronisation

## Übersicht

Dieser Skill überträgt die Tagesplanung aus Dimacon nach Clockin. Für ein gegebenes Datum
werden alle geplanten Termine aus Dimacon geladen und als Projekte in Clockin angelegt oder
aktualisiert — inklusive Adressdaten, Kundenzuordnung und Mitarbeiter-Zuweisungen.

**Standard-Datum:** Heute (aktuelles Datum), es sei denn der Benutzer gibt ein anderes Datum an.

---

## Schritt 1: Datum bestimmen

Wenn der Benutzer kein Datum nennt, wird das heutige Datum verwendet.
Bei Angaben wie "morgen", "nächsten Montag", "für den 25.03." das entsprechende Datum im
ISO-8601-Format berechnen (YYYY-MM-DD).

---

## Schritt 2: Alle Termine für das Datum aus Dimacon laden

Verwende `miranum:dimacon_get_appointments_by_period` mit `from` und `to` auf das gleiche Datum gesetzt.

```
miranum:dimacon_get_appointments_by_period(from: "2026-03-23", to: "2026-03-23")
```

Das liefert alle Termine über alle Teams hinweg. Jeder Termin enthält:
- `jobId` — der zugehörige Auftrag
- `teamId` — das zugeordnete Team
- `date` — Termin-Zeitpunkt (in UTC)

Wenn keine Termine gefunden werden, den Benutzer informieren dass für dieses Datum keine
Aufträge geplant sind.

---

## Schritt 3: Job-Details, Projekte, Kunden und Team-Zuweisungen laden

Für jeden gefundenen Termin die vollständigen Daten sammeln. Das geht am effizientesten
wenn man die Aufrufe parallelisiert:

### 3a: Job-Details laden
Für jeden eindeutigen `jobId` → `miranum:dimacon_get_job(jobId)`.
Das liefert das zugehörige `projectId`, `customerId` und die `teamAssignments` mit den
konkreten `employeeId`-Zuordnungen für das Datum.

### 3b: Projekt-Details laden
Für jedes eindeutige `projectId` → `miranum:dimacon_get_project(projectId)`.
Liefert: Projektname, Straße, PLZ+Ort.

### 3c: Kunden-Details laden
Für jede eindeutige `customerId` → `miranum:dimacon_get_customer(customerId)`.
Liefert: Kundennummer (z.B. "10025"), Kundenname.

Rufe so viele Anfragen wie möglich parallel auf, um Zeit zu sparen.

---

## Schritt 4: Mitarbeiter-Mapping Dimacon → Clockin

Die Team-Zuweisungen aus Schritt 3a enthalten `employeeId` (Dimacon-UUID).
Über die Dimacon-Mitarbeiterliste (`miranum:dimacon_list_employees`) den vollen Namen und die
E-Mail-Adresse ermitteln.

Dann in Clockin den entsprechenden Mitarbeiter über eine **mehrstufige Suche** zuordnen:

### Stufe 1: Nachname
```
miranum:clockin_search_employees(scopes: [{"name": "byLastName", "parameters": ["<Nachname>"]}])
```
Wenn **genau 1 Treffer** → Zuordnung gefunden.

### Stufe 2: Vorname (bei mehreren Treffern)
Wenn die Nachname-Suche **mehrere Treffer** liefert (z.B. zwei Mitarbeiter mit Nachname
"Müller"), die Ergebnisse nach `first_name` filtern. Der Mitarbeiter, dessen Vorname mit
dem Dimacon-Vornamen übereinstimmt, wird zugeordnet.

### Stufe 3: E-Mail (bei doppeltem Vor- und Nachname)
Wenn auch nach Vorname+Nachname noch **mehrere Treffer** übrig sind, die Ergebnisse
zusätzlich nach E-Mail-Adresse filtern (`email`-Feld in Clockin vs. E-Mail aus Dimacon).

### Kein eindeutiger Treffer
Wenn keine der Stufen einen eindeutigen Mitarbeiter liefert, den Benutzer informieren
und die manuelle Zuordnung erfragen — nicht raten.

### Bekannte Mitarbeiter-IDs (Cache)

| Dimacon-Name         | Dimacon-ID                             | Clockin-ID |
|----------------------|----------------------------------------|------------|
| Dennis Quathamer     | 29c184f2-af9a-4268-80bc-642f6065e820   | 656        |
| Jendrik Lucas        | c897389a-27a3-40b7-bdfa-0c8fd0e83d13   | 127273     |
| Marius Vogeler       | 167d7a17-b9f1-4c2e-996e-0b2f76b270cd   | 371385     |
| Björn Müller         | 2c5f71ba-887e-422b-bd39-1bd06e270906   | 51778      |
| Aaron-Philip Ihmels  | e6391ebe-9a49-48eb-bc80-451f1d6a8bae   | (nachschlagen) |

Verwende diese Tabelle als Schnellzugriff. Falls ein Mitarbeiter nicht in der Tabelle steht,
die mehrstufige Suche (Nachname → Vorname → E-Mail) durchführen.

---

## Schritt 5: Kunden-Mapping Dimacon → Clockin (mit Lexware-Abgleich)

Die Dimacon-Kundennummer (z.B. "10025") wird in Clockin als `identifier` gespeichert.
Suche den Kunden in Clockin:
```
miranum:clockin_search_customers(scopes: [{"name": "byNameOrNumber", "parameters": ["<Kundennummer>"]}])
```

### 5a: Kunde in Clockin gefunden → fertig, weiter.

### 5b: Kunde NICHT in Clockin gefunden → Lexware prüfen

Wenn der Kunde in Clockin nicht existiert, in Lexware suchen:
```
miranum:lexoffice_list-contacts(filters: [{"property": "company_name", "value": "<Kundenname>"}])
```
Alternativ über die Kundennummer suchen, falls diese in Lexware als `customer_number` hinterlegt ist.

### 5c: Kunde auch NICHT in Lexware → Kunden in Lexware anlegen

Mit den Daten aus Dimacon (`miranum:dimacon_get_customer`) den Kunden in Lexware erstellen:
```
miranum:lexoffice_create-contact(
  version: 0,
  roles: { customer: { number: <nächste freie Kundennummer> } },
  company: { name: "<Kundenname aus Dimacon>" },
  addresses: { billing: [{
    street: "<Straße>",
    zip: "<PLZ>",
    city: "<Ort>",
    country_code: "DE"
  }] },
  email_addresses: { business: ["<E-Mail falls vorhanden>"] },
  phone_numbers: { business: ["<Telefon falls vorhanden>"] }
)
```

### 5d: Dimacon-Kundennummer abgleichen

Wenn der Kunde in Lexware gefunden wurde, prüfen ob die Kundennummer in Dimacon mit der
Lexware-Kundennummer übereinstimmt. Falls die Nummern abweichen, die Dimacon-Kundennummer
auf die Lexware-Nummer aktualisieren — **Lexware ist die Quelle der Wahrheit**:
```
miranum:dimacon_update_customer(
  customerId: "<Dimacon-Kunden-UUID>",
  customerNumber: "<Lexware-Kundennummer>"
)
```

### 5e: Kunden in Clockin anlegen

Sobald der Kunde in Lexware existiert (entweder gefunden oder neu erstellt) und die
Kundennummer in Dimacon abgeglichen ist, den Kunden auch in Clockin anlegen:
```
miranum:clockin_create_customer(
  name: "<Kundenname>",
  number: "<Lexware-Kundennummer>",
  street: "<Straße>",
  zip: "<PLZ>",
  city: "<Ort>"
)
```

Die Lexware-Kundennummer wird als `number` in Clockin verwendet, damit alle drei Systeme
(Dimacon, Lexware, Clockin) über dieselbe Kundennummer verknüpft sind.

### Bekannte Kunden-IDs (Cache)

| Kundenname                    | Dimacon-Nr | Clockin-ID |
|-------------------------------|------------|------------|
| Nieters Haustechnik GmbH      | 10025      | 433864     |
| BPT Betonprüftechnik Weser-Ems| 10065      | 433806     |
| Langer E-Technik GmbH         | 10207      | (nachschlagen) |
| Minimax GmbH                  | 10284      | (nachschlagen) |

---

## Schritt 6: Projekte in Clockin prüfen (KRITISCH!)

**Vor dem Anlegen eines Projekts IMMER prüfen, ob es bereits in Clockin existiert!**

Die Dimacon-Projekt-UUID wird als `number` (Projektnummer) in Clockin gespeichert. Suche:
```
miranum:clockin_search_projects(scopes: [{"name": "byNumber", "parameters": ["<dimacon-project-id>"]}])
```

Dieser Schritt ist entscheidend, weil:
- Projekte in Clockin können schon von früheren Tagen existieren (z.B. mehrtägige Baustellen)
- Doppelte Projekte verursachen Verwirrung bei der Zeiterfassung
- Auf bestehende Projekte wurden möglicherweise schon Stunden gebucht

### Ergebnis: Projekt existiert NICHT → Neu anlegen (Schritt 7)
### Ergebnis: Projekt existiert bereits → Aktualisieren (Schritt 8)

---

## Schritt 7: Neues Projekt in Clockin anlegen

```
miranum:clockin_create_project(
  name: "<Projektname aus Dimacon>",
  number: "<Dimacon-Projekt-UUID>",
  customer_id: <Clockin-Kunden-ID>,
  destination_street: "<Straße>",
  destination_zip: "<PLZ>",
  destination_city: "<Ort>",
  contact_name: "<Ansprechpartner falls bekannt>",
  start_date: "<Datum>T07:30:00+01:00"
)
```

### Zeitkonvertierung
Dimacon speichert Termine in UTC. Für die Clockin-Startzeit:
- **Winterzeit (letzter Sonntag Oktober bis letzter Sonntag März):** UTC + 1h (CET)
- **Sommerzeit (letzter Sonntag März bis letzter Sonntag Oktober):** UTC + 2h (CEST)

Das ISO-8601-Format mit Offset (`+01:00` oder `+02:00`) sorgt dafür, dass Clockin die
richtige Uhrzeit anzeigt.

### Adressdaten aufbereiten
Die Dimacon-Projekte liefern `zipCity` als kombinierten String (z.B. "49565 Bramsche").
Für Clockin aufteilen in `destination_zip` und `destination_city`.

Nach dem Anlegen: Die neue Clockin-Projekt-ID merken für die Mitarbeiter-Zuweisung.

---

## Schritt 8: Bestehendes Projekt aktualisieren

Wenn das Projekt bereits existiert, werden **sowohl das Startdatum als auch die
Mitarbeiter-Zuweisungen** aktualisiert:

### 8a: Startdatum aktualisieren
```
miranum:clockin_update_project(
  project: <Clockin-Projekt-ID>,
  start_date: "<Datum>T07:30:00+01:00"
)
```

### 8b: Mitarbeiter-Zuweisungen synchronisieren
Erst prüfen, welche Mitarbeiter aktuell in Clockin zugewiesen sind:
```
miranum:clockin_list_project_employees(project: <Clockin-Projekt-ID>)
```

Dann mit den Dimacon-Team-Zuweisungen für das Datum abgleichen:
- **Fehlende Mitarbeiter hinzufügen:** Wenn ein Mitarbeiter in Dimacon zugewiesen ist
  aber nicht in Clockin → mit `miranum:clockin_attach_project_employees` hinzufügen
- **Überschüssige Mitarbeiter entfernen:** Wenn ein Mitarbeiter in Clockin zugewiesen
  ist aber laut Dimacon heute NICHT an dem Projekt arbeitet →
  mit `miranum:clockin_detach_project_employees` entfernen

So spiegelt Clockin immer die aktuelle Tagesplanung aus Dimacon wider.

---

## Schritt 9: Mitarbeiter zuweisen (bei neuen Projekten)

Alle Mitarbeiter aus den Dimacon-Team-Zuweisungen zuordnen:
```
miranum:clockin_attach_project_employees(
  project: <Clockin-Projekt-ID>,
  resources: [<Clockin-Mitarbeiter-ID-1>, <Clockin-Mitarbeiter-ID-2>]
)
```

---

## Schritt 10: Nicht eingeplante Projekte archivieren

Nach der Synchronisation alle aktiven Clockin-Projekte archivieren, die am Zieldatum
**NICHT in Dimacon eingeplant** sind. So bleibt die Clockin-Projektliste sauber und
zeigt nur Projekte an, die tatsächlich am jeweiligen Tag anstehen.

### Vorgehen:
1. Alle aktiven (nicht archivierten) Projekte in Clockin laden:
   ```
   miranum:clockin_search_projects(scopes: [{"name": "unarchived"}])
   ```
   Ggf. über mehrere Seiten paginieren.

2. Liste der **synchronisierten Projekt-IDs** aus den Schritten 7-9 erstellen
   (alle Projekte, die für das Zieldatum in Dimacon eingeplant waren).

3. Jedes aktive Clockin-Projekt, das **NICHT** in dieser Liste steht → archivieren:
   ```
   miranum:clockin_update_project(project: <Clockin-Projekt-ID>, archived: true)
   ```

4. Archivierte Projekte in der Zusammenfassung auflisten.

**Wichtig:** Es werden ALLE nicht eingeplanten Projekte archiviert — nicht nur solche
ohne Mitarbeiter. Das stellt sicher, dass in Clockin nur die Tagesplanung des
Zieldatums sichtbar ist.

---

## Schritt 11: Zusammenfassung ausgeben

Nach Abschluss der Synchronisation dem Benutzer eine Übersicht zeigen:

| Projekt | Clockin-ID | Mitarbeiter | Status |
|---------|-----------|-------------|--------|
| AOK Bramsche | 1215876 | Dennis Quathamer | Neu erstellt |
| Schulte Haus 1 | 1207290 | Jendrik Lucas | Aktualisiert |

Unterscheide klar zwischen:
- **Neu erstellt** — Projekt wurde neu angelegt
- **Aktualisiert** — Startdatum/Mitarbeiter wurden aktualisiert
- **Unverändert** — Projekt existierte bereits und brauchte keine Änderung
- **Archiviert** — Projekt hatte keine Mitarbeiter mehr und wurde archiviert

---

## Hinweise und häufige Fehler

- **Immer erst prüfen, dann anlegen!** Doppelte Projekte in Clockin sind problematisch.
  Wenn versehentlich Duplikate entstehen: nur die neuen löschen (mit `force: true`),
  nie die alten, auf die schon Zeit gebucht wurde.
- **`miranum:dimacon_get_current_jobs` zeigt nicht alle Termine!** Diese Funktion liefert nur
  eine begrenzte Auswahl. Verwende stattdessen `miranum:dimacon_get_appointments_by_period` um
  wirklich alle Termine eines Tages über alle Teams zu bekommen.
- **Clockin-API Typen beachten:** Die `project`-Parameter bei Clockin-Aufrufen wie
  `miranum:clockin_update_project`, `miranum:clockin_delete_project`, `miranum:clockin_list_project_employees`,
  `miranum:clockin_attach_project_employees` erwarten eine Zahl (number), keine Strings.
- **`miranum:clockin_search_customers` Scope:** Heißt `byNameOrNumber` (nicht `byNumber`).
- **Mitarbeiter-Zuweisung bei bestehenden Projekten:** Immer vollständig synchronisieren —
  fehlende hinzufügen UND überschüssige entfernen, damit Clockin die aktuelle
  Dimacon-Tagesplanung widerspiegelt.
- **Parallele API-Aufrufe:** Wo möglich, Aufrufe parallelisieren (z.B. alle Job-Details
  gleichzeitig laden). Das spart deutlich Zeit bei vielen Terminen.
