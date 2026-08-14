# 02 — Wann merkt die App, dass ein neuer Abschnitt begonnen hat?

Type: research
Status: open
Blocked by: —
Map: [map.md](../map.md)

## Question

Die Farbfläche muss über Stunden unbeaufsichtigt korrekt bleiben. Wie löst die
App den Wechsel zum nächsten aktiven Abschnitt zuverlässig aus?

Konkret zu klären:

- **Intervall oder Timeout?** Jede Sekunde/Minute pollen, oder ein `setTimeout`
  exakt auf die nächste Startzeit setzen? Wie verlässlich sind lange
  JS-Timer in React Native, wenn die App stundenlang im Vordergrund idlet?
- **Bildschirm aus per Sperrtaste**: die App bleibt im Vordergrund, aber das
  Display ist aus. Laufen JS-Timer weiter, oder werden sie gedrosselt? Was
  sieht der Nutzer beim Wiedereinschalten — sofort die richtige Farbe oder
  kurz die alte?
- **AppState-Wechsel**: Muss beim Zurückkehren in den Vordergrund die aktive
  Farbe neu berechnet werden? (Vermutlich ja — die Antwort soll das bestätigen
  oder widerlegen.)
- **Sommer-/Winterzeit**: Was passiert in der Nacht, in der eine Stunde
  übersprungen oder wiederholt wird? Verhält sich ein auf 03:00 gesetzter
  Timeout dann falsch?
- **Systemzeit-Sprünge**: Nutzer stellt die Uhr, oder die Zeitzone wechselt.
  Gibt es dafür ein Event, oder fällt das unter regelmäßiges Nachrechnen?

Das Ergebnis soll eine begründete Empfehlung für **einen** Mechanismus sein,
plus die Liste der Fälle, gegen die die Unit-Tests der reinen Zeitfunktion
antreten müssen.
