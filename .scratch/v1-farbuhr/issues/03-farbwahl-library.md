# 03 — Welche Farbwahl-Library?

Type: research
Status: open
Blocked by: —
Map: [map.md](../map.md)

## Question

Welche Library liefert unter Expo 56 mit Dev Client einen Farbwähler für iOS und
Android?

Bewertungsmaßstab, in dieser Reihenfolge:

- **Läuft sie überhaupt** unter Expo 56 / React Native in der dort gesetzten
  Version, und wird sie noch gepflegt?
- **Native Abhängigkeit oder pures JS?** Beides ist zulässig (Dev Client), aber
  pures JS hält den Build einfacher.
- **Passt sie zum Anwendungsfall**: gewählt werden kräftige, gut
  unterscheidbare Vollflächen-Farben plus sehr dunkle Töne für die Nacht. Ein
  Wähler, der auf Pastell-Nuancen optimiert ist, ist hier eher schlechter.
- Braucht es überhaupt eine Library, oder reicht eine selbstgebaute Palette aus
  festen Farbfeldern? Diese Option ausdrücklich mitbewerten — sie könnte
  gewinnen.

Ergebnis: eine Empfehlung mit Zweitplatziertem und der Begründung, warum. Die
Frage, *wie sich die Auswahl anfühlt*, gehört nicht hierher — die entscheidet
Ticket 07.
