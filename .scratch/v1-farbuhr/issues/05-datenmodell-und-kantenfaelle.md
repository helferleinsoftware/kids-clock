# 05 — Datenmodell des Plans und seine Kantenfälle

Type: grilling
Status: resolved
Blocked by: —
Map: [map.md](../map.md)

## Question

Die Ring-Semantik steht (siehe [CONTEXT.md](../../../CONTEXT.md)). Offen ist die
genaue Form eines Abschnitts und das Verhalten des Plans an seinen Rändern.

Zu entscheiden:

- **Form eines Abschnitts**: Was gehört hinein außer Name, Farbe und Startzeit?
  Braucht ein Abschnitt eine stabile ID, oder ist die Startzeit der Schlüssel?
  (Das entscheidet, ob man die Startzeit eines bestehenden Abschnitts überhaupt
  bearbeiten kann, oder nur löschen und neu anlegen.)
- **Leerer Plan**: Der Erststart liefert Vorgaben mit — aber der Nutzer kann
  alle löschen. Welche Farbe zeigt die Fläche dann? Ist ein leerer Plan
  überhaupt erlaubt, oder verweigert die App das Löschen des letzten
  Abschnitts?
- **Ein einziger Abschnitt**: Der Ring besteht aus einem Element, die Farbe gilt
  24 Stunden. Ist das ein zulässiger Zustand?
- **Zwei Abschnitte mit gleicher Startzeit**: verhindern, oder zulassen und eine
  Reihenfolge festlegen?
- **Sortierung**: Sortiert sich der Plan immer automatisch nach Startzeit, oder
  gibt es eine vom Nutzer bestimmte Reihenfolge? (Bei Ring-Semantik spricht
  alles für automatisch — die Frage ist, ob das beim Bearbeiten überrascht,
  wenn ein Eintrag unter dem Finger wegspringt.)
- **Name**: Pflicht oder optional? Wo ist er überhaupt sichtbar, wenn die
  Farbfläche nichts als Farbe zeigt — nur in den Settings? Falls ja: wozu dient
  er dann, und rechtfertigt das seine Existenz in v1?
- **Persistenz in AsyncStorage**: unter welchem Schlüssel, in welcher Form, und
  bekommt der gespeicherte Stand von Anfang an eine Versionsnummer für spätere
  Migrationen?

Diese Antworten fixieren die Datenstruktur, gegen die die Unit-Tests der reinen
Zeitfunktion geschrieben werden.

## Answer

Ermittelt in drei Grilling-Runden. Kurzfassung: **ein Abschnitt hat keine
eigene ID — seine Startzeit _ist_ seine Identität**, der Plan ist eine nach
Startzeit eindeutige Liste, und der leere Plan ist ein legaler Zustand, den ein
Settings-Gate abfängt statt ihn zu verbieten.

### Die Datenstruktur

```ts
type Segment = {
  startMinute: number; // 0…1439, Minute des Tages
  name: string;        // Pflicht, getrimmt nicht leer
  color: string;       // "#RRGGBB", Großbuchstaben, kein Alpha
};

type Plan = Segment[]; // eindeutig nach startMinute, Reihenfolge bedeutungslos
```

Drei Felder, kein viertes. Alles im Code und im Speicher ist **englisch**
(`Segment`, `Plan`, `activeSegment`, `minuteOfDay`); deutsch bleiben die
Prosa-Artefakte — Tickets, Karte, CONTEXT.md, Commits. Die deutschen Begriffe
bleiben die Leitwährung, CONTEXT.md führt die Zuordnung.

### Die Entscheidungen im Einzelnen

**Identität = Startzeit, keine ID.** Der Auslöser war die Regel für doppelte
Startzeiten: Speichern auf eine belegte Startzeit **ersetzt** den bestehenden
Abschnitt. Damit verhält sich der Plan wie eine Map mit `startMinute` als
Schlüssel — eine UUID daneben wäre eine zweite Identität ohne Aufgabe, denn
zwei Abschnitte mit gleicher Startzeit und verschiedener UUID sind nach dieser
Regel derselbe Abschnitt. Folge: doppelte Startzeiten sind strukturell
unmöglich, ein Tie-Break in der reinen Funktion wird nicht gebraucht.

Preis dafür, bewusst gezahlt: „Zeit von 19:00 auf 19:30 ändern" ist alten
Schlüssel löschen + neuen schreiben, der Bearbeiten-Screen muss also die
**ursprüngliche** Startzeit mitführen. Und React-`key`s in der Liste wechseln,
wenn eine Zeit geändert wird — kosmetisch, kein Zustandsverlust, solange der
Bearbeiten-Screen einen eigenen Entwurf hält.

**Startzeit als eine Zahl `0…1439`**, nicht als `{ stunde, minute }`. Das ist
exakt die Eingabe der reinen Funktion aus Ticket 02 — keine Umrechnung zwischen
Speicher und Kern, sondern nur am Picker-Rand (Ticket 04 liefert ein `Date`,
gespeichert wird `getHours() * 60 + getMinutes()`). Eine Validierungsregel statt
zweier, Vergleich und Sortierung sind eine Subtraktion. Preis: das gespeicherte
JSON liest sich für Menschen schlechter (`1140` statt `19:00`) — akzeptiert,
niemand liest diese Datei von Hand.

**Ersetzen passiert still, ohne Rückfrage.** Der scharfe Fall ist nicht das
Neuspeichern desselben Abschnitts, sondern: 19:00 öffnen, Zeit auf 07:00
stellen — der bestehende 07:00-Abschnitt ist weg. Der Plan hat eine Handvoll
Zeilen, die Liste steht direkt hinter dem Speichern sichtbar da, und ein
Bestätigungsdialog wäre die einzige Modal-Ebene der ganzen App. **Beobachtungs-
punkt für Ticket 06**: falls sich das im Prototyp falsch anfühlt, ist es dort
billig nachzurüsten.

**Name ist Pflicht, getrimmt nicht leer.** Keine Vorbelegung, keine
Zeichengrenze, kein Möbel drumherum. Sichtbar ist er ausschließlich in den
Settings — die Farbfläche zeigt nach wie vor nichts als Farbe.

**Farbe als `#RRGGBB`**, Großbuchstaben, kein Alpha, **nie als Palettenindex**.
Ein Index würde jede spätere Palettenänderung rückwirkend auf bestehende Pläne
durchschlagen lassen; der Hex-String ist selbsttragend und überlebt sowohl eine
neue Palette als auch den in Ticket 03 offengehaltenen Rückfall auf
`reanimated-color-picker`. Validierung ist ein Regex, kein Farbraum-Parser.

**Sortierung ist abgeleitet, nie gespeichert.** Gespeichert wird eine
unsortierte Liste, sortiert wird beim Lesen: **aufsteigend ab 00:00**, nicht ab
dem aktiven Abschnitt. Bei Ring-Semantik ist eine abweichende Nutzerreihenfolge
bedeutungslos — die Zeit _ist_ die Reihenfolge, alles andere wäre eine zweite
Wahrheit, die auseinanderlaufen kann. Die reine Funktion sortiert selbst und
setzt keine sortierte Eingabe voraus (Ticket 02, Testfall A8).

**Der leere Plan ist erlaubt — und ersetzt den Erststart-Entwurf.** Kein
Minimum, keine verweigerte Löschung des letzten Abschnitts. Stattdessen: **mit
0 Abschnitten kann der Settings-Screen nicht verlassen werden**, ab 1 Abschnitt
geht es zur Farbfläche. Das vereinfacht auch den Erststart — die App liefert
**keine** Vorgabe-Abschnitte mehr mit, sie startet mit leerer Liste in den
Settings, und der Settings-Screen _ist_ das Onboarding. Damit ist die frühere
Karten-Entscheidung „App liefert 19:00 rot / 07:00 grün mit" **aufgehoben**.
Ein Plan mit genau einem Abschnitt bleibt zulässig und bedeutet per Definition:
immer dieselbe Farbe, 24 Stunden lang.

Unabhängig von der UI-Regel bleibt die reine Funktion **total**: bei leerem Plan
liefert `activeSegment` `null` und wirft nicht (Ticket 02, Testfall A10). Der
leere Plan kann sie über kaputte gespeicherte Daten trotzdem erreichen.

**Keine Obergrenze für die Anzahl Abschnitte.** Aus der Eindeutigkeit folgt
bereits eine natürliche Grenze von 1440, die nie erreicht wird. Eine ausgedachte
Grenze müsste man erklären, durchsetzen und irgendwann zurücknehmen. Wie sich
eine lange Liste auf dem Screen verhält, ist eine reine Darstellungsfrage für
Ticket 06.

### Persistenz

**Ein Schlüssel, ein Dokument, Version im Dokument — nicht im Schlüsselnamen.**

- Schlüssel: `kids-clock/plan` (fest, für immer)
- Inhalt:

```json
{
  "version": 1,
  "segments": [
    { "startMinute": 1140, "name": "Schlafen", "color": "#B03030" },
    { "startMinute": 420, "name": "Aufstehen", "color": "#2E9E4F" }
  ]
}
```

Ein Schlüssel pro Abschnitt scheidet aus: er bringt Teilschreibungen und damit
halb gespeicherte Pläne, ein Dokument wird atomar geschrieben. Die Version
gehört **ins** Dokument, weil eine spätere Migration die alten Daten sonst nur
findet, wenn sie den alten Schlüsselnamen errät.

Die Liste gewinnt gegen eine Objekt-Map `{"1140": {…}}`, weil JSON-Objekt-
schlüssel immer Strings sind — die Map handelt sich Parsen und Rückkonvertieren
bei jedem Laden ein, plus einen Sonderfall für `"0"`. Eindeutigkeit ist dafür
eine Validierungsregel statt einer Struktureigenschaft, was durch die
Alles-oder-nichts-Regel unten folgenlos bleibt.

**Kaputter oder fremder Stand: alles verwerfen, mit leerem Plan starten.** Das
ganze Dokument wird beim Laden validiert; bei _jedem_ Fehler — unparsbares JSON,
unbekannte `version`, ungültiges Feld, doppelte `startMinute` — startet die App
mit leerem Plan. Keine Teilrettung: die würde einen Plan erzeugen, den der
Nutzer nie eingegeben hat, ohne dass er es merkt. Der Fehlerfall ist genau
deshalb billig, weil der leere Plan legal ist — kaputte Daten landen im
Settings-Screen mit Gate, also im selben Zustand wie ein Erststart, ohne einen
einzigen zusätzlichen Fehlerpfad.

### Was das für Ticket 02 bedeutet

Die Testfallliste aus Ticket 02 bleibt gültig, zwei Zeilen bekommen jetzt ihre
Antwort, und die Validierung kommt als eigene Gruppe dazu:

- **A9** (zwei Abschnitte mit identischer Startzeit) wandert von der Logik in
  die Validierung: die reine Funktion kann den Fall nicht mehr sehen, das
  Dokument wird beim Laden komplett verworfen.
- **A10** (leerer Plan) bleibt in der Logik: `activeSegment(plan, m) → null`,
  kein Throw — die Farbfläche wird in diesem Zustand ohnehin nicht angezeigt.
- Neue Gruppe **D — `parsePlan(raw)`**: gültiges Dokument; unparsbares JSON;
  `version` fehlt/unbekannt; `startMinute` fehlt/kein Integer/-1/1440;
  `color` ohne `#`/mit Alpha/3-stellig; `name` fehlt/leer/nur Leerzeichen;
  doppelte `startMinute`; leeres `segments`-Array (gültig! → leerer Plan);
  unbekannte Zusatzfelder (→ ignorieren, kein Fehler). Erwartung in allen
  Fehlerfällen: leerer Plan, kein Throw.

### Was offen an Ticket 06 geht

- Sortiert sich die Liste **live** beim Bearbeiten um, oder erst nach dem
  Bestätigen? (Der Eintrag, der unter dem Finger wegspringt.)
- Fühlt sich das stille Ersetzen richtig an, oder braucht es doch eine
  Rückfrage?
- Wie sieht das Gate aus, das den Settings-Screen bei 0 Abschnitten
  festhält — deaktivierter Zurück-Weg, Hinweistext, beides?
