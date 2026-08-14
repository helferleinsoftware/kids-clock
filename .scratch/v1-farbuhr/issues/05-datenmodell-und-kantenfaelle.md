# 05 — Datenmodell des Plans und seine Kantenfälle

Type: grilling
Status: open
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
