---
name: planung
description: >
  Plant neue Aufträge in Dimacon ein – vom Kundenabgleich über Projektauswahl
  bis zur Job- und Working-Erstellung. Verwende diesen Skill IMMER wenn der
  Benutzer einen Auftrag einplanen, anlegen, erstellen oder terminieren will.
  Auch verwenden bei: "Plane einen Auftrag für [Kunde]", "Leg einen Job an",
  "Neuer Auftrag für [Projekt]", "Termin einplanen", "Auftrag erstellen",
  "Job anlegen für [Datum]", "Kernbohrung einplanen", "Wandsäge-Auftrag",
  "Plane das ein", "Kannst du das einplanen?", "Auftrag aus Angebot erstellen",
  "Mach einen Job draus", "Plane [Angebotsnummer] ein". Triggert auch wenn
  der Benutzer eine Lexware-Angebotsnummer (z.B. "AG-2026-042") nennt und
  einen Auftrag daraus machen will, oder wenn er einen Kunden + Datum nennt
  und etwas eingeplant haben möchte. NICHT verwenden für Abrechnung (→ dimacon:abrechnung)
  oder Angebotserstellung aus Planungsexport-URLs (→ dimacon:lexware).
---

# Dimacon Auftragsplanung

Dieser Skill plant neue Aufträge (Jobs) in Dimacon ein. Er führt den Benutzer
durch den Prozess: Kunde finden → Projekt auswählen → Job anlegen → optional
Workings vorbereiten.

---

## 0. Eingangsprüfung – Was will der Benutzer?

Bevor du loslegst, kläre welcher Fall vorliegt:

### Fall A: Auftrag aus Lexware-Angebot
Der Benutzer nennt eine **Angebotsnummer** (z.B. "AG-2026-042", "Angebot 42")
oder sagt "Plan das Angebot ein".

→ Angebot in Lexware suchen:
```
miranum:lexoffice_list-voucherlist(voucherType="quotation", voucherNumber=<nummer>)
```
→ Angebotsdaten lesen: `miranum:lexoffice_get-quotation(id=<uuid>)`
→ Kundenname und Positionen aus dem Angebot extrahieren
→ Weiter mit Schritt 1 (Kunde suchen), Kundenname aus Angebot übernehmen

### Fall B: Freier Auftrag
Der Benutzer beschreibt was gemacht werden soll, nennt einen Kunden und/oder
ein Projekt. Kein Lexware-Angebot involviert.

→ Direkt zu Schritt 1

### Im Zweifel: Frag den Benutzer
Wenn unklar ist ob ein Angebot referenziert wird oder nicht, frag kurz nach:
"Gibt es ein bestehendes Angebot in Lexware dazu, oder soll ich einen freien
Auftrag anlegen?"

---

## 1. Kunde finden

Kundennamen aus dem Kontext ermitteln (aus Angebot, Benutzereingabe, oder
vorherigem Gespräch).

```
miranum:dimacon_list_customers()
```

Lokal nach dem genannten Kundennamen filtern (Teilstring-Match, case-insensitive).

**Ergebnis:**
- **1 Treffer** → `customerId` merken, weiter zu Schritt 2
- **Mehrere Treffer** → Dem Benutzer die Optionen zeigen, auswählen lassen
- **Kein Treffer** → Benutzer fragen: "Kunde '[Name]' nicht gefunden. Soll ich
  den Kunden in Dimacon anlegen?" Falls ja → `miranum:dimacon_create_customer` (Daten
  vom Benutzer erfragen: Name, Straße, PLZ/Ort, ggf. Telefon/E-Mail)

---

## 2. Projekt auswählen

Projekte des Kunden finden:

```
miranum:dimacon_list_projects()
```

Lokal filtern: alle Projekte mit `customerId` des gefundenen Kunden.

**Ergebnis:**
- **Projekte vorhanden** → Dem Benutzer die Liste zeigen. Wenn der Benutzer
  bereits einen Projektnamen / eine Adresse genannt hat, automatisch matchen.
  Sonst auswählen lassen.
- **Keine Projekte** → Benutzer fragen ob ein neues Projekt angelegt werden
  soll. Falls ja:
  ```
  miranum:dimacon_create_project(
    name=<Projektname>,
    customerId=<customerId>,
    street=<Straße>,
    zipCity=<PLZ Ort>,
    customAttributeValues=[]
  )
  ```
  Projektname, Adresse und sonstige Daten beim Benutzer erfragen.

---

## 3. Termin und Team klären

Bevor der Job angelegt wird, brauchen wir:

### 3a. Datum
- Aus Benutzereingabe ableiten ("nächsten Montag", "am 25.03.", "diese Woche")
- Datum in ISO 8601 umwandeln: `YYYY-MM-DDT07:00:00.000+01:00` (Standardstart 07:00)
- Winter (Nov–Mär): `+01:00`, Sommer (Apr–Okt): `+02:00`

### 3b. Team
```
miranum:dimacon_list_active_teams()
```
Dem Benutzer die verfügbaren Teams zeigen und auswählen lassen.
Wenn der Benutzer bereits ein Team/einen Mitarbeiter namentlich genannt hat,
automatisch matchen.

### 3c. Angebotsnummer (optional)
Falls ein Lexware-Angebot vorliegt (Fall A): die Angebotsnummer als
`offerNumber` im Job hinterlegen. Das ist wichtig für die spätere Abrechnung
(→ dimacon:abrechnung, Pfad A).

---

## 4. Job anlegen

```
miranum:dimacon_create_job(
  projectId=<projectId>,
  dueDate=<datum ISO 8601>,
  appointments=[
    {
      date: <datum ISO 8601>,
      teamId: <teamId>
    }
  ],
  automaticallyCreateGuestAccess=true,
  customAttributeValues=[],
  offerNumber=<angebotsnummer oder leer>,
  description=<Beschreibung der Arbeiten>
)
```

### Pflichtfelder
- `projectId` – aus Schritt 2
- `dueDate` – Zieldatum
- `appointments` – mindestens 1 Termin mit `date` + `teamId`
- `automaticallyCreateGuestAccess` – immer `true`
- `customAttributeValues` – leeres Array `[]` falls keine CAs

### Optionale Felder
- `offerNumber` – Lexware-Angebotsnummer (Fall A)
- `description` – Beschreibung der Arbeiten
- `additionalInformation` – Zusatzinfos
- `color` – Farbmarkierung
- `invoiceNumber` – Rechnungsnummer (selten bei Neuanlage)
- `scheduledDateConfirmed` – ob Termin bestätigt ist

### Mehrere Termine
Wenn der Benutzer mehrere Tage plant (z.B. "Montag und Dienstag"), für jeden
Tag einen eigenen Appointment im Array anlegen. Jeder Appointment braucht
ein eigenes `date` und `teamId` (kann dasselbe Team sein).

---

## 5. Workings vorbereiten (optional)

Workings sind die einzelnen Tätigkeiten eines Auftrags (Kernbohrungen,
Wandsägeschnitte etc.). Sie werden normalerweise VOR ORT vom Team dokumentiert,
können aber vorausgefüllt werden wenn die Arbeiten bekannt sind.

### Wann Workings anlegen?
- **Fall A (Angebot vorhanden):** Frag den Benutzer: "Soll ich die Positionen
  aus dem Angebot als Workings voranlegen?" Falls ja → Workings aus den
  Angebotspositionen ableiten.
- **Fall B (Freier Auftrag):** Nur wenn der Benutzer explizit Workings will
  oder die konkreten Leistungen beschreibt.
- **Im Zweifel:** Nicht automatisch anlegen – das Team macht das vor Ort.

### Working-Templates ermitteln
```
miranum:dimacon_list_working_templates()
```
Die Templates bestimmen welche Attribute pro Working erfasst werden.
Typische Templates:
- **Kernbohrung** (DOING) – Attribute: Durchmesser, Tiefe, Material, Aufwand
- **Wandsäge** (DOING) – Attribute: Schnittlänge, Wandstärke, Material
- **Nebentätigkeit** (ADDITIONAL_DOING) – Attribute: Dauer, Beschreibung

### Working erstellen
```
miranum:dimacon_create_working(
  jobId=<jobId>,
  templateId=<templateId>,
  type="DOING",
  amount=1,
  appointment=<appointmentId>,
  attributes=[
    { templateAttributeId: "<attrId>", value: "<wert>" }
  ],
  fileNames=[]
)
```

Oder für mehrere auf einmal:
```
miranum:dimacon_create_workings(
  jobId=<jobId>,
  workings=[...]
)
```

Die `appointmentId` kommt aus der Job-Erstellung (Schritt 4 – Response enthält
die erstellten Appointments mit IDs).

---

## 6. Zusammenfassung

Nach erfolgreicher Erstellung dem Benutzer eine kurze Zusammenfassung geben:

```
✅ Auftrag angelegt:
   Kunde:   <Kundenname>
   Projekt: <Projektname>
   Termin:  <Datum>, Team: <Teamname>
   Angebot: <Angebotsnr.> (falls vorhanden)
   Workings: <Anzahl> vorausgefüllt (falls erstellt)
```

---

## Regeln

### IMMER
- **Kunde zuerst** suchen – nie blind eine customerId annehmen
- **Projekt zum Kunden** – nie ein Projekt eines anderen Kunden verwenden
- **Team auswählen lassen** – nie ein Team raten
- **Termin bestätigen** – bevor der Job erstellt wird, den Benutzer die
  Details bestätigen lassen
- **`automaticallyCreateGuestAccess=true`** – damit Kunden den Bericht sehen

### NIE
- Nicht automatisch Workings anlegen ohne Rückfrage
- Nicht Jobs ohne Appointment erstellen
- Nicht `miranum:dimacon_list_jobs` oder `miranum:dimacon_get_current_jobs` verwenden –
  wir ERSTELLEN hier, wir suchen keine bestehenden Jobs
- Nicht die Abrechnung starten – dafür ist `dimacon:abrechnung` zuständig

### DATENVALIDIERUNG
- Datumsformat immer ISO 8601 mit Zeitzone
- `customAttributeValues` immer als Array senden (auch wenn leer: `[]`)
- Team-IDs und Projekt-IDs sind UUIDs (Strings)
- Angebotsnummern aus Lexware exakt übernehmen

---

## Fehlerbehandlung

| Situation | Aktion |
|-----------|--------|
| Kunde nicht gefunden | Benutzer fragen ob Neuanlage |
| Kein Projekt beim Kunden | Benutzer fragen ob Neuanlage |
| Kein aktives Team | Benutzer informieren |
| Job-Erstellung fehlgeschlagen | Fehler zeigen, ggf. Daten korrigieren |
| Angebot in Lexware nicht gefunden | Benutzer informieren, als freien Auftrag weiter |
| Benutzer bricht ab | Kein Problem, nichts wird erstellt bis zur Bestätigung |
