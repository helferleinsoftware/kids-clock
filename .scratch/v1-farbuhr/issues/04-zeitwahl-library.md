# 04 — Welche Zeitwahl-Library?

Type: research
Status: open
Blocked by: —
Map: [map.md](../map.md)

## Question

Womit stellt der Nutzer die Startzeit eines Abschnitts ein — unter Expo 56 mit
Dev Client, auf iOS und Android?

Konkret zu klären:

- Ist `@react-native-community/datetimepicker` weiterhin der Standardweg unter
  Expo 56, und wie sehen die beiden Plattform-Oberflächen tatsächlich aus?
- Gibt es relevante Alternativen, und was gewinnt man mit ihnen?
- **Nur Uhrzeit, keine Datumsauswahl** — welche Optionen erzwingen das sauber?
- **24-Stunden- vs. 12-Stunden-Anzeige**: folgt der Wähler der
  Systemeinstellung, und lässt sich das überschreiben?
- Wie wird der Wähler dargestellt — inline, Modal, Bottom Sheet — und was davon
  ist plattformabhängig? Diese Antwort füttert direkt Ticket 06.

Ergebnis: eine Empfehlung plus eine kurze Beschreibung, wie der Wähler auf
beiden Plattformen konkret erscheint.
