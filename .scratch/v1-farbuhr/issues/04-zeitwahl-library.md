# 04 — Welche Zeitwahl-Library?

Type: research
Status: resolved
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

## Answer

### Empfehlung

**`@react-native-community/datetimepicker` in der von Expo 56 gepinnten Version
9.1.0, immer mit `mode="time"`.** Konkret:

- **Android**: die imperative API `DateTimePickerAndroid.open({ mode: 'time',
  value, is24Hour: true, onValueChange, onDismiss })`, ausgelöst durch Tippen
  auf die Abschnitts-Zeile. Das ist auch die von der Library selbst empfohlene
  Variante auf Android.
- **iOS**: die Komponente `<RNDateTimePicker mode="time" display="compact" />`
  direkt in der Zeile platzieren — sie ist dort die Zeit-Kachel, iOS öffnet das
  Rad selbst als Popover. Kein eigener Modal-Code auf iOS nötig.
- Für den dunklen Settings-Screen (Ticket 06) auf iOS zusätzlich
  `themeVariant="dark"` setzen, sonst richtet sich die Textfarbe des Wählers
  nach dem Systemthema und nicht nach dem Screen-Hintergrund.

**Zweite Wahl mit klarem Umschaltkriterium:**
`@expo/ui/community/datetime-picker`. Umsteigen, wenn der Prototyp in Ticket 06
eines von beidem will: (a) einen **inline** eingebetteten Zeitwähler *auf
Android* (kann die Community-Library grundsätzlich nicht), oder (b) die
**Material-3-Optik** bzw. zur Laufzeit gefärbte Dialoge ohne Eingriff in
`styles.xml`. Der Umstieg kostet einen Import und ein paar Prop-Namen — die APIs
sind absichtlich fast deckungsgleich. Diese Entscheidung darf also getrost bis
nach dem Prototyp warten.

### Ausgangslage Expo 56 (geprüft)

- Expo 56 = React Native **0.85.3**, React **19.2.3** (Template
  `expo-template-blank-typescript@56.0.33`).
- `packages/expo/bundledNativeModules.json` auf Branch `sdk-56` pinnt beide
  Kandidaten: `@react-native-community/datetimepicker` → **9.1.0** und
  `@expo/ui` → **~56.0.25**. Beide werden also von `npx expo install`
  versionsrichtig gezogen, sind von Expo für dieses SDK getestet, brauchen kein
  Config-Plugin und je einen neuen Dev Build.
- Die Community-Library ist gepflegt: 9.0.0 und 9.1.0 am 16./17.03.2026, MIT,
  Fabric/TurboModule-Support (`codegenConfig` im Paket vorhanden). Sie ist damit
  weiterhin der Standardweg.
- Neu in SDK 56: Expo liefert einen eigenen **Drop-in-Ersatz**
  (`@expo/ui/community/datetime-picker`, SwiftUI auf iOS, Jetpack Compose auf
  Android). Die Expo-Doku-Seite der Community-Library verweist prominent darauf,
  **erklärt sie aber nicht für veraltet**.
- Ab 9.x ist `onChange` deprecated; neu sind `onValueChange` und `onDismiss`.
  Beim Schreiben der Spec direkt die neuen Namen verwenden.

### Kandidatenvergleich

| | `@react-native-community/datetimepicker` 9.1.0 | `@expo/ui` 56.0.25 (`community/datetime-picker`) |
| --- | --- | --- |
| Von Expo 56 gepinnt | ja | ja |
| Nur Uhrzeit | `mode="time"` — sauber, auf Android sogar ein eigener nativer Dialog | `mode="time"` — sauber, iOS `displayedComponents: ['hourAndMinute']`, Android Material-3-`TimePicker` |
| Inline auf iOS | ja | ja |
| **Inline auf Android** | **nein** (Komponente rendert `null` und öffnet einen Dialog) | **ja** (`presentation="inline"`) |
| Android-Optik out of the box | AppCompat-Dialog (klassisch), weil Expos App-Theme `Theme.AppCompat.DayNight.NoActionBar` ist | Material 3, unabhängig vom App-Theme |
| Material 3 erzwingbar | nur mit `design="material"` **und** App-Theme `Theme.Material3.…` → eigenes Config-Plugin + Prebuild | Standard |
| Farben zur Laufzeit | iOS `accentColor`/`textColor`/`themeVariant`; Android nur über mitgeliefertes Config-Plugin + Prebuild | `accentColor`, `elementColors` (sehr feingranular), iOS `themeVariant` |
| `minuteInterval` (z. B. 5er-Schritte) | ja (iOS nur bei `display="spinner"`) | **nicht unterstützt** |
| Imperative API | ja (`DateTimePickerAndroid.open`) | nein, rein deklarativ |
| Reife | seit Jahren im Einsatz, sehr breit getestet | jung; Pfad wurde erst im 56er-Zyklus verschoben, allein in 56.0.x vier Layout-/Größen-Bugs gefixt, und das Paket liefert auch in Patch-Releases noch „Breaking changes" |

### So erscheint der Wähler konkret

**iOS — nie ein eigener Dialog.** Beide Libraries liefern eine *View*, die man
selbst platziert (Zeile, Modal, Sheet). Was man sieht, steuert `display`:

- `display="default"` → `UIDatePicker`-Stil `.automatic`, auf dem iPhone löst das
  zu `.compact` auf: eine kleine graue, abgerundete Kachel mit „20:00". Tippen
  öffnet ein System-Popover mit dem Stundenrad — die Optik aus Wecker und
  Einstellungen.
- `display="compact"` → dasselbe, nur explizit.
- `display="spinner"` → das klassische Rad direkt im Layout, zwei Spalten
  (Stunde | Minute), bei 12-Stunden-Anzeige plus AM/PM-Spalte. Braucht ordentlich
  vertikalen Platz.
- `display="inline"` → iOS-14+-Variante; bei `mode="time"` faktisch ebenfalls eine
  Rad-Darstellung.
- Es gibt **kein OK/Abbrechen**: jede Drehung meldet sofort einen Wert.
- Beim `@expo/ui`-Ersatz ist das Mapping `default→automatic`, `compact→compact`,
  `inline→graphical`, `spinner→wheel`; ohne `title` wird `.labelsHidden()`
  gesetzt, man sieht also nur die Kachel bzw. das Rad ohne Label-Zeile.

**Android — immer ein Dialog, außer man nimmt `@expo/ui` inline.**

- Community-Library: die Komponente rendert `null` und öffnet den nativen
  `TimePickerDialog`. Bild: modaler Dialog mit Überschrift („Uhrzeit wählen"),
  großes `HH:MM`-Feld oben, darunter das Zifferblatt (bei 24 h mit Außenring
  00–23), links unten das Tastatur-Symbol zum Umschalten auf Direkteingabe,
  rechts unten „Abbrechen / OK". Der Wert kommt **erst bei OK**
  (`onValueChange`), „Abbrechen" bzw. Zurück feuert `onDismiss`. Optik ist der
  AppCompat-Dialog, solange Expos Standard-App-Theme unverändert bleibt.
- `@expo/ui` mit `presentation="dialog"` (Android-Default): derselbe Ablauf, aber
  Material-3-Look und zur Laufzeit einfärbbar. Achtung: der Dialog öffnet **beim
  Mounten**, man montiert/demontiert ihn also selbst.
- `@expo/ui` mit `presentation="inline"` und `mode="time"`: Material-3-`TimePicker`
  im Layout, `TimePickerLayoutType.Vertical` (Zeitfeld oben, Zifferblatt
  darunter), **ohne OK-Knopf** — jede Änderung meldet sofort. Das ist der einzige
  Weg zu einem eingebetteten Zeitwähler auf Android. `display="spinner"` wird in
  diesem Fall ignoriert; es gibt inline nur das Zifferblatt.

**Nur Uhrzeit, kein Datum** — bei beiden sauber erzwungen: `mode="time"` zeigt
auf keiner Plattform irgendeine Datumsauswahl, auf Android ist es sogar ein
anderer nativer Dialog. Aber: der Wert bleibt technisch ein volles `Date`. Der
Datumsanteil ist der, den man in `value` hineingibt. Für Ticket 05 heißt das:
Stunde/Minute speichern, für den Wähler ein Wegwerf-`Date` bauen, aus dem
Ergebnis wieder nur Stunde/Minute lesen.

**24 h vs. 12 h**

- Android, Community-Library: folgt per Default der Systemeinstellung
  (`DateFormat.is24HourFormat(context)`), `is24Hour={true|false}` überschreibt.
- Android, `@expo/ui`: der Kotlin-Default ist `is24Hour = true`, der Wähler zeigt
  also **immer 24 h**, solange man nicht explizit `false` übergibt. Für dieses
  Projekt praktisch, aber es ist eben *nicht* „folgt dem System".
- iOS, beide: **kein Prop**. Der Wähler folgt Gerät/Locale (Einstellungen →
  Allgemein → Datum & Uhrzeit → 24-Stunden-Format). Auf einem deutschen Gerät
  also 24 h. Ein Erzwingen ist nur über das iOS-`locale`-Prop denkbar, das die
  Library selbst als unzuverlässig außerhalb von `display="spinner"`
  kennzeichnet — nicht darauf bauen.
- Unabhängig davon: die **Anzeige der Startzeit in der Liste** formatieren wir
  selbst (`Intl`/`toLocaleTimeString` mit `hourCycle: 'h23'`). Dort ist
  Konsistenz vollständig in unserer Hand, egal was der Wähler tut.

### Geprüft und verworfen

- **`react-native-date-picker` 5.0.13** (letztes Release 05.06.2025): eingebautes
  Modal *oder* inline, auf beiden Plattformen dieselbe Rad-Optik, `mode="time"`,
  New Architecture seit 4.3.0. Reizvoll wegen der plattformgleichen Darstellung,
  aber: nicht in Expos `bundledNativeModules`, kein Release seit RN 0.80,
  Kompatibilität mit RN 0.85 unbelegt. Nur nehmen, wenn *identische* Optik auf
  beiden Plattformen eine harte Anforderung wird.
- **`react-native-modal-datetime-picker` 18.0.0** (Aug 2024): der klassische
  Wrapper für „plattformübergreifendes Modal um die Community-Library". Seit zwei
  Jahren kein Release, Peer-Range `>=6.7.0`, während 9.x die Event-API umgestellt
  hat. Nicht empfehlenswert.
- **Eigenbau aus zwei Rädern** (`@react-native-picker/picker` 2.11.4, in SDK 56
  gepinnt, oder `react-native-wheel-pick`): erzwingt „nur Uhrzeit" trivialerweise
  und sieht überall gleich aus, baut aber nach, was das System schon kann, und
  verliert Lokalisierung und Bedienungshilfen, die man sonst geschenkt bekommt.
  Nur, falls der Prototyp bewusst etwas Nicht-Standardmäßiges will (z. B. sehr
  große, nachttaugliche Räder).

### Was das für Ticket 06 heißt

Zwei anschauliche Varianten, beide mit der empfohlenen Library baubar:

1. **Zeile öffnet Systemwähler** — auf iOS die Compact-Kachel in der Zeile
   (Popover mit Rad), auf Android Tippen auf die Zeile → Material-Dialog mit
   OK/Abbrechen. Wenig Code, überall vertraut, aber die beiden Plattformen fühlen
   sich unterschiedlich an, und Android hat einen Bestätigungsschritt, iOS nicht.
2. **Eigener Bearbeiten-Screen/Modal mit großem Rad** — auf beiden Plattformen
   `display="spinner"` bzw. inline, plus eigene „Fertig"-Zeile. Einheitlicher,
   nachttauglicher (große Trefferflächen), und `minuteInterval={5}` wird hier
   nutzbar, weil es auf iOS nur im Spinner greift. Kostet einen dritten Screen
   oder ein Modal.

Variante 2 ist der Grund, warum `minuteInterval` oben als Unterschied auftaucht:
5-Minuten-Schritte gibt es nur mit der Community-Library und auf iOS nur im
Spinner.

### Nicht verifiziert

- **Nichts davon lief auf einem Gerät oder Simulator.** Alle Aussagen stammen aus
  Quellcode, Doku-Quelldateien und npm-Metadaten.
- `docs.expo.dev` und `expo.dev` sind in dieser Session vom Egress-Proxy
  blockiert, `github.com` ebenfalls (nur `raw.githubusercontent.com` und
  `registry.npmjs.org` waren erreichbar). Gelesen wurden daher die Doku-Quellen im
  Repo `expo/expo` (Branches `sdk-56`/`main`), der Kotlin-/Swift-/TS-Quellcode von
  `expo-ui` sowie die npm-Tarballs beider Pakete. **Issue-Tracker konnte ich
  nicht einsehen** — offene Bugs zu beiden Libraries sind also ungeprüft.
- Wie `display="inline"` (SwiftUI `.graphical`) bei `mode="time"` auf iOS genau
  aussieht, konnte ich nur aus der Prop-Dokumentation ableiten, nicht sehen.
- Ob das iOS-`locale`-Prop 24 h tatsächlich gegen die Systemeinstellung erzwingt:
  ungeprüft, die Library rät davon ab.
- Häufig berichtet, hier nicht bestätigt: die Compact-Kachel auf iOS braucht
  gelegentlich eine explizite Breite bzw. `alignSelf`, sonst kollabiert sie. Für
  `@expo/ui` gibt es dazu einen Fix im 56er-Zyklus (Host `matchContents`); für die
  Community-Library auf RN 0.85 ist es unverifiziert. Beim Prototyp darauf achten.
- Dass 9.1.0 auf RN 0.85 rund läuft, schließe ich nur daraus, dass Expo genau
  diese Version für SDK 56 pinnt.
