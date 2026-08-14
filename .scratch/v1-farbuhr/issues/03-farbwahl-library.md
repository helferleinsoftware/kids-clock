# 03 — Welche Farbwahl-Library?

Type: research
Status: resolved
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

## Answer

### Empfehlung

**Keine Library. Eine selbstgebaute Palette aus festen Farbfeldern gewinnt.**

Zweitplatzierter: **`reanimated-color-picker` 5.1.2** — die einzige Farbwähler-
Library, die unter Expo 56 heute nachweislich lebt. Sie ist der Rückfallweg,
falls Ticket 07 doch freie Farbwahl statt fester Felder will.

Ausdrücklich *nicht* empfohlen: `@expo/ui`-`ColorPicker` (nur iOS, kein
Android-Gegenstück) und sämtliche älteren Picker-Pakete (seit 2019–2024 tot).

### Vergleich

Basis: Expo SDK 56 zieht laut `bundledNativeModules.json` aus `expo@56.0.19`
(publiziert 2026-08-06) **react-native 0.85.3, react-native-reanimated 4.3.1,
react-native-gesture-handler ~2.31.1, react-native-worklets 0.8.3,
AsyncStorage 2.2.0**. Reanimated 4 heißt: nur New Architecture. Das
Default-Template (`expo-template-default@56.0.33`) enthält Reanimated,
Gesture-Handler und Worklets **bereits** — Reanimated-basierte Libraries
kosten hier also keine zusätzliche Native-Abhängigkeit.

| Kandidat | Version | Letzter Publish | Native / JS | Wartungssignal | iOS + Android |
| --- | --- | --- | --- | --- | --- |
| **Eigene Palette, keine Library** | — | — | pures JS (`View`/`Pressable`) | entfällt — kein Fremdcode | ja |
| **`reanimated-color-picker`** | 5.1.2 | 2026-07-01 | pures JS; peer: reanimated ≥2, gesture-handler ≥2, expo ≥44 | aktiv: v5.0.0 am 2026-05-31, 479 Sterne, 4 offene Issues, DevDeps gegen RN 0.85.3 / React 19.2 | ja |
| `@expo/ui` `ColorPicker` (`swift-ui`) | 56.0.25 | 2026-08-06 | **nativ** (SwiftUI), Modul ist über `expo-router` ohnehin im Baum | first-party, `@expo/ui` seit SDK 56 stabil | **nein — nur iOS** |
| `react-native-wheel-color-picker` | 1.3.1 | 2024-01-01 | JS, zieht `react-native-elevation@^1.0.0` | tot (2 Releases in 5 Jahren) | ja, ungetestet |
| `react-native-color-picker` (instea) | 0.6.0 | 2020-09-03 | JS | tot | ja, ungetestet |
| `react-native-color-picker-ios` | 0.1.3 | 2024-04-17 | **nativ** (UIColorPicker) | faktisch tot | nein — nur iOS |
| `react-native-color-picker-input` | 0.1.3 | 2026-05-22 | JS, peer `react-native-svg` | 2 Releases, 1 Maintainer, keine Historie | unbekannt |
| `@darthrapid/react-native-color-picker` | 1.2.1 | 2026-08-04 | JS, peer `react-native-svg` | 4 Releases seit Feb 2026, 1 Maintainer | unbekannt |
| `react-native-reanimated-color-picker` (iyegoroff), `react-native-hsv-color-picker`, `react-native-color-wheel`, `react-native-color-panel`, `react-native-slider-color-picker` | 0.0.11 / 1.0.2 / 0.1.7 / 1.0.1 / 2.2.6 | 2021 / 2021 / 2019 / 2019 / 2023 | JS | tot; teils peer auf `react-native-image-filter-kit` bzw. `expo-linear-gradient@^8` | nein |

### Begründung

1. **Das Bewertungskriterium „passt zum Anwendungsfall" schlägt alles andere.**
   Gebraucht werden wenige, kräftige, weit auseinanderliegende Vollflächen-
   Farben plus sehr dunkle Nachttöne. Ein kontinuierlicher HSV-Wähler
   optimiert genau die Gegenrichtung: Er macht Pastelltöne und
   Nachbar-Nuancen bequem erreichbar und die *bewusst grobe* Auswahl mühsam.
   Sehr dunkle Töne trifft man auf einem Rad nur, indem man den
   Helligkeits-Slider fast auf null zieht — schlecht reproduzierbar und
   ausgerechnet im relevanten Bereich am ungenauesten.

2. **Die Library-Komponente, die zum Fall passt, ist der triviale Teil.** In
   `reanimated-color-picker` wäre das `Swatches` — nachgelesen im Tarball
   5.1.2: rund 30 Zeilen, ein `View` mit `Pressable`-Kacheln und
   `backgroundColor`, Default-Palette ist die Material-500-Reihe. Um sie zu
   benutzen, braucht man trotzdem den `<ColorPicker>`-Context-Wrapper
   drumherum. Man zöge also die ganze Library für ihr simpelstes Stück
   herein und verlöre dabei die Kontrolle über genau das, was hier zählt:
   welche Farben in der Liste stehen.

3. **Die Kosten der Library liegen alle in den Teilen, die man nicht
   bräuchte.** `lib/` ist 4,3 MB, davon 532 KB PNG-Gradienten für Rad und
   Panels. Die offenen Issues hängen ebenfalls dort: #82 „screen freeze/crash
   when color wheel loads or re-renders" (offen seit Juli 2024), #105
   CircularHue rendert nach Rotation gelegentlich falsch (offen seit Mai
   2026), #101 „onChangeJS causes the app to crash on android" (Jan 2026).
   Für eine App, die nachts durchläuft, ist „Wheel-Panel kann einfrieren"
   ein schlechter Tausch für Funktionalität, die man nicht will.

4. **An der Native-Frage scheitert die Library nicht** — Reanimated 4.3.1 und
   Gesture-Handler 2.31.1 sind im Expo-56-Template ohnehin gesetzt. Sie
   scheitert an Kriterium 3 (Anwendungsfall) und 4 (braucht es sie
   überhaupt).

5. **Der eine native Weg, der reizvoll wäre, ist plattform-halbiert.**
   `@expo/ui` ist seit SDK 56 stabil und über `expo-router` sogar schon als
   transitive Dependency da. Sein `ColorPicker` existiert aber ausschließlich
   unter `@expo/ui/swift-ui`, mit `@platform ios` annotiert und
   `ios/ColorPickerView.swift` als einziger Implementierung — im
   Jetpack-Compose-Zweig und im `universal`-Export ist er nicht vorhanden
   (im Tarball 56.0.25 geprüft). Für iOS+Android bräuchte man also sowieso
   einen zweiten, selbstgebauten Android-Zweig — und dann kann man gleich
   nur den bauen.

6. **Was die eigene Palette gewinnt:** null Installationen, null
   Build-Risiko, identisches Verhalten auf beiden Plattformen, keine
   Kopplung an das Reanimated-4-Upgrade-Fenster, und die Farben sind eine
   getestete Datenkonstante statt einer Benutzereingabe. Dunkle Nachttöne
   werden schlicht als eigene Felder in die Liste aufgenommen, statt sie
   erdrosseln zu müssen. Persistenz bleibt ein Hex-String in AsyncStorage.

7. **Das Rückfahrticket ist billig.** Beide Wege speichern denselben Wert
   (ein Hex-String pro Abschnitt). Sollte Ticket 07 zu dem Schluss kommen,
   dass feste Felder nicht reichen, ist der Wechsel auf
   `reanimated-color-picker` eine lokale Änderung im Settings-Screen und
   kein Datenmodell-Bruch. Umgekehrt gilt das genauso.

Anmerkung für Ticket 07, nicht hier zu entscheiden: Anzahl, Anordnung und
konkrete Hexwerte der Felder, sowie ob es neben den Feldern eine
„Sonstige"-Tür gibt. Falls dort später Schieberegler auftauchen sollen —
`@react-native-community/slider` 5.2.0 ist in SDK 56 mitgeliefert.

### Nicht verifiziert

- **Kein Build, kein Laufzeittest.** Auftrag war reine Recherche. Die
  Verträglichkeit von `reanimated-color-picker` 5.1.2 mit Reanimated 4.3.1
  ist *statisch* geprüft: die Library importiert aus Gesture-Handler nur
  `Gesture` / `GestureDetector` / `GestureHandlerRootView` (moderne API) und
  aus Reanimated nur `runOnJS`, `runOnUI`, `useSharedValue`,
  `useAnimatedStyle`, `useAnimatedProps`, `useAnimatedRef`,
  `useDerivedValue`, `withTiming` — alle in 4.3.1 noch exportiert.
  `useAnimatedGestureHandler` kommt nicht vor. Einziger Fund:
  `Animated.addWhitelistedNativeProps` wird an zwei Stellen benutzt und ist
  in Reanimated 4 als deprecated No-Op deklariert — kein Absturz, aber die
  `PreviewText`/`InputWidget`-Vorschau könnte dadurch stumpf sein. Ungetestet.
- **Download-Zahlen fehlen.** `api.npmjs.org` ist vom Egress-Proxy blockiert,
  ebenso die GitHub-API. Sterne (479) und offene Issues (4) stammen aus der
  gerenderten GitHub-Seite, nicht aus der API.
- **`expo.dev` ist blockiert.** Die SDK-56-Versionsangaben stammen deshalb
  nicht aus dem Changelog, sondern aus `bundledNativeModules.json` im
  npm-Tarball von `expo@56.0.19` — belastbarer, aber eben eine andere Quelle.
- **Issue #101** (`onChangeJS`-Crash auf Android) — offen oder geschlossen
  ließ sich nicht eindeutig klären.
- **Zukunft von `@expo/ui`**: dass unter SDK 56 kein Android-`ColorPicker`
  existiert, ist am Tarball 56.0.25 belegt. Ob Expo einen in SDK 57+
  nachliefert, wurde nicht geprüft (SDK 57 ist inzwischen `latest`, für
  dieses Projekt aber nicht gesetzt).
- Die beiden 2026er-Newcomer (`react-native-color-picker-input`,
  `@darthrapid/react-native-color-picker`) wurden nur anhand ihrer
  npm-Metadaten beurteilt, nicht anhand ihres Codes.
