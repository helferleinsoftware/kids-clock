# 06 — Wie sieht der Settings-Screen aus und wie fühlt er sich an?

Type: prototype
Status: open
Blocked by: 04, 05
Map: [map.md](../map.md)

## Question

Eine der zwei Geschmacksfragen dieser Karte. Reden reicht hier nicht — es
braucht etwas Anschaubares zum Draufzeigen.

Zu klären:

- **Der Weg hinein und hinaus**: Long-Press auf der Farbfläche öffnet die
  Settings. Wie kommt man zurück? Gibt es beim Long-Press eine sichtbare
  Rückmeldung, oder springt der Screen einfach um?
- **Die Liste**: Wie wird ein Abschnitt in der Zeile dargestellt — Farbfeld,
  Startzeit, Name? Was ist das dominante Element? Sieht man der Liste an, dass
  sie ein Ring über den Tag ist, oder liest sie sich wie eine beliebige Liste?
- **Bearbeiten**: Direkt in der Zeile, oder über einen Detail-Screen? Wie legt
  man einen Abschnitt an, wie löscht man einen — Swipe, Button, Edit-Modus?
- **Nachtbedienung**: Realistischer Fall — jemand steht um 22:00 im dunklen
  Kinderzimmer und ändert eine Zeit. Sollte der Settings-Screen dunkel sein,
  auch wenn die Farbfläche gerade hell leuchtet?
- **Zwei Screens per expo-router** ist gesetzt. Offen ist, ob das Bearbeiten
  eines einzelnen Abschnitts ein dritter Screen wird oder ein Modal.

Nachgereicht aus [Ticket 05](05-datenmodell-und-kantenfaelle.md):

- **Das Gate bei 0 Abschnitten**: Mit leerem Plan kann der Settings-Screen nicht
  verlassen werden, und genau so sieht der Erststart aus. Deaktivierter
  Zurück-Weg, Hinweistext, ein leerer Zustand mit Aufforderung — oder alles
  zusammen?
- **Umsortieren beim Bearbeiten**: Die Liste ist immer aufsteigend ab 00:00
  sortiert. Sortiert sie sich **live** um, während die Zeit verstellt wird, oder
  erst nach dem Bestätigen? (Der Eintrag, der unter dem Finger wegspringt.)
- **Stilles Ersetzen**: Eine Zeit auf eine bereits belegte Startzeit zu stellen,
  ersetzt den bestehenden Abschnitt ohne Rückfrage. Fühlt sich das im Prototyp
  richtig an, oder braucht es doch eine Bestätigung?
- **Lange Liste**: Es gibt keine Obergrenze für die Anzahl Abschnitte. Wie
  verhält sich die Liste, wenn sie länger wird als der Screen?

Vorgehen: mit `/prototype` ein bis zwei Wegwerf-Varianten bauen, als Asset vom
Ticket verlinken, und **den Nutzer wählen lassen**. Nicht selbst entscheiden.

Blockiert, weil die Darstellung der Zeit-Eingabe (Ticket 04) und die Frage, was
ein Abschnitt überhaupt enthält (Ticket 05), das Layout bestimmen.
