# 08 — Der Home Indicator lässt sich auf iOS nicht dauerhaft verstecken. Was nun?

Type: grilling
Status: open
Blocked by: —
Map: [map.md](../map.md)
Aufgetaucht durch: [Ticket 01](01-vollbild-und-wachhalten.md)

## Question

Die ursprüngliche v1-Anforderung lautet „keine Status Bar, keine Home
Indicators". Ticket 01 hat ergeben, dass die zweite Hälfte auf iOS **nicht
erfüllbar** ist: UIKit kennt nur *auto*-hide. Der Indikator blendet sich nach
Inaktivität aus und kommt bei **jeder** Berührung zurück. Ein dauerhaftes
Verstecken existiert schlicht nicht.

Für diese App heißt das konkret: die Farbfläche ist über Nacht sauber, aber in
dem Moment, in dem das Kind den Bildschirm berührt, erscheint der Strich am
unteren Rand — und mit ihm die Möglichkeit, per Wischen die App zu verlassen.

Zu entscheiden:

- **Reicht auto-hide?** Wenn der Indikator ohnehin nur bei Berührung sichtbar
  wird und danach wieder verschwindet, ist das Problem vielleicht kleiner als
  es klingt.
- **Ist das Verlassen der App das eigentliche Problem?** Nicht der sichtbare
  Strich stört, sondern dass ein Kind aus der Ampel herauswischen und im
  Homescreen landen kann. Falls ja, ist die Frage nicht „wie verstecke ich den
  Indikator", sondern „wie halte ich die App im Vordergrund" — eine andere
  Frage mit anderen Antworten.
- **Geführter Zugriff / App-Pinning als Teil des Setups?** iOS „Geführter
  Zugriff" und Android „App-Pinning" sperren das Gerät wirklich auf eine App.
  Beides sind Geräteeinstellungen, keine App-Funktionen — die App kann sie
  nicht anschalten. Akzeptierst du das als dokumentierten Einrichtungsschritt
  („einmal pro Gerät aktivieren"), oder gehört Kiosk-Verhalten gar nicht zu
  v1?
- **Android**: dort verstecken sich die Leisten wirklich, aber ein Randwisch
  zeigt sie kurz — auch das ist seit SDK 56 nicht mehr konfigurierbar. Gilt
  dieselbe Antwort für beide Plattformen, oder darf sich das unterscheiden?

Ergebnis: eine klare, bewusst getroffene Erwartung an v1 — und falls Geführter
Zugriff dazugehört, die Notiz, dass die Einrichtungsanleitung Teil der Spec
wird.
