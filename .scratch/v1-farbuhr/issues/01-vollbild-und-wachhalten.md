# 01 — Vollbild ohne System-Chrome und Wachhalten in Expo 56

Type: research
Status: resolved
Blocked by: —
Map: [map.md](../map.md)

## Question

Wie erreicht eine Expo-56-App mit Dev Client auf **iOS und Android**, dass die
Farbfläche wirklich die ganze Fläche einnimmt und dauerhaft sichtbar bleibt?

Konkret zu klären:

- **Status Bar** verstecken — dauerhaft, nicht nur bis zur ersten Berührung.
- **Home Indicator** auf iOS ausblenden bzw. so weit zurücknehmen wie das OS es
  zulässt. Was ist die tatsächliche Grenze — verschwindet er ganz, oder nur nach
  Inaktivität und kehrt bei Berührung zurück?
- **Android Navigation Bar** und Immersive Mode: welcher Weg ist unter Expo 56
  der aktuelle, und wie zuverlässig hält er, wenn der Nutzer an den Rand wischt?
- **Edge-to-edge** unter Android: ist es in Expo 56 erzwungen, und was heißt das
  für eine Fläche, die unter den Systemleisten durchlaufen soll?
- **Sleep verhindern**: `expo-keep-awake` oder ein anderer Weg. Gilt das auch,
  wenn die App stundenlang unberührt im Vordergrund steht?
- Welche dieser Punkte brauchen **native Konfiguration** (Config Plugin,
  `app.json`, Info.plist / Manifest) und sind damit nur im Dev Build wirksam?

Gesucht sind Fakten und ein konkreter Umsetzungsvorschlag pro Plattform, inklusive
der Stellen, an denen die Plattform *nicht* mitspielt — die gehören genauso ins
Ergebnis, weil sie die Produkterwartung verschieben.
