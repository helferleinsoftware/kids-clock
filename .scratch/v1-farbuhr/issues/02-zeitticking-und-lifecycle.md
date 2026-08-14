# 02 — Wann merkt die App, dass ein neuer Abschnitt begonnen hat?

Type: research
Status: resolved
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

## Answer

### Empfehlung: pollen und stateless neu rechnen — kein Timeout auf die Startzeit

**Mechanismus (einer, verbindlich):**

1. **Reine Funktion ohne `Date`.** `aktiverAbschnitt(plan, minuteDesTages) → Abschnitt`,
   wobei `minuteDesTages` eine ganze Zahl `0..1439` ist. Die Funktion kennt weder
   Datum noch Zeitzone noch Sommerzeit. Sie ist total, deterministisch und
   zustandslos — insbesondere setzt sie **nicht** voraus, dass die Eingabe
   monoton wächst.
2. **Dünner Adapter am Rand.** `minuteDesTages(new Date())` =
   `d.getHours() * 60 + d.getMinutes()`. Nur lokale Getter, nie UTC-Getter, nie
   ein aus Komponenten *konstruiertes* `Date`.
3. **Tick = sich selbst neu planender `setTimeout`**, nie `setInterval` mit
   fester Länge und nie ein langer Timeout auf die nächste Startzeit. Delay =
   „Zeit bis zur nächsten vollen Minute" + ~50 ms Epsilon, geklemmt auf
   `[250 ms, 60 s]`. Jeder Tick liest die Uhr **frisch** und rechnet den
   Abschnitt komplett neu. Zustand wird nur geschrieben, wenn sich der Abschnitt
   tatsächlich ändert.
4. **`AppState`-Listener**: bei Übergang nach `'active'` sofort neu rechnen und
   den Timeout abbrechen + neu planen (Phase resynchronisieren).
5. Tick und Listener leben **oberhalb** der expo-router-Screens (Root-Layout /
   Hook / kleiner Store), damit der Weg in die Settings und zurück nichts
   abreißt.
6. Verboten: einen Index „aktueller Abschnitt" mitführen und ihn beim Tick
   „eins weiterschalten", oder aus einer alten Zeit + Delta auf die neue
   schließen. Beides bricht bei Rücksprüngen (Herbst-Zeitumstellung,
   Uhr zurückgestellt, Bildschirm 9 h aus).

**Warum nicht der exakte `setTimeout` auf die nächste Startzeit:** Er ist in
genau den Fällen falsch, wegen denen dieses Ticket existiert, und er ist auf
den beiden Plattformen *unterschiedlich* falsch (siehe F3/F4). Er muss außerdem
eine Wanduhrzeit in eine Dauer umrechnen — und diese Umrechnung ist an der
Frühjahrsumstellung um exakt eine Stunde daneben. Ein Fehler von einer Stunde
in einer Aufsteh-Ampel ist der Totalschaden des Produkts.

**Warum nicht `setInterval(1000)`:** funktioniert korrektheitsmäßig genauso
(auch selbstkorrigierend), weckt den JS-Thread aber 86 400× pro Tag statt
1 440× und bringt keine Genauigkeit, die etwas nützt — Startzeiten haben
Minutenauflösung. Falls die Minutengrenzen-Arithmetik in der Umsetzung als
Risiko empfunden wird, ist `setInterval(1000)` mit *identischer*
Neuberechnung ein zulässiger, nur teurerer Rückfall; die Korrektheit hängt
nicht am Takt, sondern am „jeder Tick rechnet komplett neu".

**Antwort auf die Ticket-Fragen in einem Satz je Punkt:**

- *Intervall oder Timeout?* Kurzer, selbst-neu-geplanter Timeout (Poll), nie
  auf die Startzeit gesetzt.
- *Bildschirm aus per Sperrtaste?* JS-Timer laufen auf **beiden** Plattformen
  nicht weiter, sie stehen (F5). Beim Wiedereinschalten feuert der Timer
  einmal — nicht nachholend N-mal (F6) — und die Farbe ist nach dem ersten
  React-Commit richtig; für einen kurzen Moment sieht der Nutzer den alten
  Frame (OS-Snapshot bzw. letzter gerenderter Frame). Das ist die einzige
  Stelle, an der die Plattform nicht mitspielt; Dauer auf Gerät messen (E1).
- *AppState-Wechsel?* **Ja**, bestätigt — neu rechnen bei `'active'`.
- *Sommer-/Winterzeit?* Für die Timer-Engines ist eine Zeitumstellung **kein**
  Sprung (F9-Vorbemerkung); falsch wäre nur die eigene Wanduhr→Dauer-Rechnung.
  Mit dem Poll-Design ist DST vollständig abgedeckt, ohne Sonderfall im Code.
- *Systemzeit-/Zeitzonensprünge?* Es gibt dafür **kein** aus RN nutzbares Event
  (F8) — fällt also unter „regelmäßig nachrechnen", wie vermutet. Ausnahme mit
  Zähnen: eine *Zeitzonen*-Änderung wirkt wegen des Hermes-Caches unter
  Umständen erst nach App-Neustart (F10).

### Die Fakten

**F0 — Was Expo 56 wirklich ist.** `expo@56.0.19` hat als devDependency
`react-native@0.85.3` und `react@19.2.3`; SDK 56 macht Hermes V1 zur
Standard-Engine, New Architecture/bridgeless ist Standard. Alle Quellcode-Aussagen
unten wurden im ausgelieferten npm-Paket `react-native@0.85.2` bzw. auf dem
Branch `0.85-stable` gelesen (Patch-Unterschied zu 0.85.3 ist für Timer irrelevant).
[Expo SDK 56 Changelog](https://expo.dev/changelog/sdk-56)

**F1 — Wo `setTimeout` in Expo 56 landet.** Bridgeless: JS → C++
[`TimerManager.cpp`](https://github.com/facebook/react-native/blob/0.85-stable/packages/react-native/ReactCommon/react/runtime/TimerManager.cpp)
→ `PlatformTimerRegistry`. iOS:
[`ObjCTimerRegistry.mm`](https://github.com/facebook/react-native/blob/0.85-stable/packages/react-native/ReactCommon/react/runtime/platform/ios/ReactCommon/ObjCTimerRegistry.mm)
→ `RCTTiming`. Android:
[`JavaTimerRegistry.h`](https://github.com/facebook/react-native/blob/0.85-stable/packages/react-native/ReactAndroid/src/main/jni/react/runtime/jni/JavaTimerRegistry.h)
→ `JavaTimerManager`. (`Libraries/Core/Timers/JSTimers.js` ist in 0.85 als
`@deprecated` markiert — der alte Bridge-Pfad; die Legacy-Doku dazu führt in die Irre.)

**F2 — iOS rechnet Timer gegen die Wanduhr.**
[`RCTTiming.mm`](https://github.com/facebook/react-native/blob/0.85-stable/packages/react-native/React/CoreModules/RCTTiming.mm):
`_target = [NSDate dateWithTimeIntervalSinceNow:targetTime]` (Z. 49),
`shouldFire:` vergleicht gegen `[NSDate date]` (Z. 57–60, 246). Ein anstehender
Timer ist also an einen **absoluten Zeitpunkt** gebunden.

**F3 — Android rechnet Timer monoton.**
[`JavaTimerManager.kt`](https://github.com/facebook/react-native/blob/0.85-stable/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/modules/core/JavaTimerManager.kt)
Z. 183: `initialTargetTime = nanoTime() / 1000000 + delay`, und
[`SystemClock.kt`](https://github.com/facebook/react-native/blob/0.85-stable/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/common/SystemClock.kt)
mappt `nanoTime()` auf `System.nanoTime()`. Ein anstehender Timer ist also an
**verstrichene Realzeit** gebunden.
→ **Konsequenz:** derselbe „langer Timeout auf 03:00"-Code verhält sich bei
einer manuell verstellten Uhr auf iOS (feuert sofort bzw. verspätet um den
Sprung) und Android (ignoriert den Sprung komplett) verschieden. Das ist der
härteste Einzelgrund gegen diesen Ansatz.

**F4 — Bildschirm aus ⇒ JS-Timer stehen, auf beiden Plattformen.**
- iOS: `RCTTiming` registriert sich auf `UIApplicationWillResignActive` /
  `DidEnterBackground` → `appDidMoveToBackground` → `stopTimers`,
  `_inBackground = YES` (Z. 139–155, 180–189). Pending Timer bekommen einen
  `NSTimer` auf den Runloop — der aber nicht bedient wird, sobald iOS den
  Prozess suspendiert: „When the app is suspended, no code within the app's
  process executes."
  ([Apple, Background Execution Sequence](https://developer.apple.com/documentation/uikit/about-the-background-execution-sequence),
  [Energy Efficiency Guide](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/WorkLessInTheBackground.html))
- Android: `JavaTimerManager` ist `LifecycleEventListener`; `onHostPause()`
  setzt `isPaused = true` und entfernt den Frame-Callback (Z. 72–76), und
  `TimerFrameCallback.doFrame` steigt bei `isPaused` sofort aus (Z. 290–293).
  Bildschirm aus ⇒ Activity `onPause`/`onStop`
  ([Activity Lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)).
  Zusätzlich ist der Choreographer vsync-getrieben — ohne Display keine Frames.
- **Doze ist hier nicht der Mechanismus**: Doze verlangt „unplugged and
  stationary"
  ([Doze und App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby)),
  und das Gerät hängt laut Karte dauerhaft am Strom. Es ist schlicht die
  pausierte Activity.

**F5 — Beim Wiedereinschalten feuert nichts nach.** Überfällige Timer werden
genau **einmal** ausgelöst, nicht N-mal: iOS `reschedule` setzt
`_target = now + interval` erst beim Feuern (Z. 65–69), Android setzt
`timer.targetTime = frameTimeMillis + timer.interval` (Z. 306). Ein
`setInterval(1000)`, das 9 Stunden stand, erzeugt beim Aufwachen **einen**
Callback und läuft dann normal weiter. Die verbreitete Sorge vor einem
„Nachhol-Sturm" ist unbegründet. Der erste Tick liegt ~1 Frame nach dem Resume;
sichtbar bleibt die alte Farbe nur bis zum ersten React-Commit danach.

**F6 — AppState wechselt beim Sperren zuverlässig.** Werte: `active`,
`background`, iOS zusätzlich `inactive` (Übergangszustand, u. a. beim
Sperren) — [RN AppState](https://reactnative.dev/docs/appstate). Android:
[`AppStateModule.kt`](https://github.com/facebook/react-native/blob/0.85-stable/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/modules/appstate/AppStateModule.kt)
Z. 41–48 mappt `onHostResume → "active"`, `onHostPause → "background"`. Auf
`'active'` neu zu rechnen ist trotz F5 nötig, weil es (a) unabhängig vom
Timer-Pfad ist, (b) die Minutenphase resynchronisiert und (c) der einzige
Haken ist, der den Fall „Uhr/Zeitzone wurde verstellt, während wir suspendiert
waren" abdeckt — und Uhr verstellen geht nur, indem man unsere App verlässt.

**F7 — Es gibt kein Uhrzeit-Änderungs-Event aus RN.** `grep` über das
komplette ausgelieferte Paket `react-native@0.85.2`: kein einziges Vorkommen
von `significantTimeChange`, `NSSystemClockDidChange`, `ACTION_TIME_CHANGED`
oder `ACTION_TIMEZONE_CHANGED`. Aus JS ist „die Systemzeit hat sich geändert"
mit RN-Bordmitteln nicht beobachtbar.
Ergänzend `expo-localization` (falls es je gebraucht wird): iOS beobachtet
`UIApplication.significantTimeChangeNotification` (feuert u. a. bei
Mitternacht, Zeitumstellung, Zeitänderung) —
[`LocalizationModule.swift` Z. 10–15](https://github.com/expo/expo/blob/sdk-56/packages/expo-localization/ios/LocalizationModule.swift);
Android feuert dasselbe Event nur aus `Application.onConfigurationChanged`
([`LocalizationPackage.kt`](https://github.com/expo/expo/blob/sdk-56/packages/expo-localization/android/src/main/java/expo/modules/localization/LocalizationPackage.kt)),
und eine Zeitzonenänderung ist auf Android *keine* Configuration-Änderung. Die
Expo-Doku sagt selbst: `getCalendars()` beim Foreground-Wechsel erneut
aufrufen, `AppState` benutzen. → asymmetrisch, als Trigger untauglich, für v1
keine Abhängigkeit wert.

**F8 — Vorbemerkung zu DST, die viel entwirrt:** Eine Sommer-/Winterzeit-Umstellung
ist **kein** Sprung der absoluten Zeit. `NSDate` und `System.nanoTime()` laufen
ungestört durch; es ändert sich nur der lokale Offset. Deshalb feuert ein auf
≤ 60 s gedeckelter Timer über die Umstellung hinweg pünktlich, und
`getHours()/getMinutes()` liefern danach die neue lokale Zeit. Falsch werden
kann nur eine selbst gerechnete Dauer „von jetzt bis 03:00 lokal" — die ist am
28.03.2027 um genau eine Stunde daneben. Genau diese Rechnung kommt im
empfohlenen Design nicht vor.

**F9 — Hermes rechnet DST ohne Neustart korrekt.** Hermes cached den
Standard-Offset einmal (`ltza_`) und den DST-Offset **pro Zeitintervall** (32
Einträge, V8-Algorithmus, max. eine Umstellung je 19 Tage); ein Zeitpunkt
außerhalb aller Intervalle löst ein frisches `localtime_r` aus.
[`DateCache.h`](https://github.com/facebook/hermes/blob/main/include/hermes/VM/JSLib/DateCache.h),
[`DateCache.cpp`](https://github.com/facebook/hermes/blob/main/lib/VM/JSLib/DateCache.cpp).
`new Date().getHours()` ist also beiderseits der Berliner Umstellung richtig,
ohne die App neu zu starten.

**F10 — Ein Zeitzonen*wechsel* ist eine andere Geschichte.** Kommentar im
Hermes-Header: „on Linux, the standard offset and DST offset is unchanged even
if TZ is updated, since the underlying time API in C library caches the time
zone… On MacOS, the time API does not cache, so we will check if the standard
offset has changed in `computeDaylightSaving()`". Hermes hat dafür
`resetTimezoneCache()` als JSI-Methode bekommen
([facebook/hermes PR #1693](https://github.com/facebook/hermes/pull/1693),
gemerged 11.06.2025) — die der Integrator aufrufen **muss**. Im ganzen Paket
`react-native@0.85.2` ist das einzige Vorkommen von `resetTimezoneCache` die
Deklaration in `ReactCommon/jsi/jsi/hermes-interfaces.h`; **niemand ruft sie
auf**. → Auf Android kann eine im Betrieb geänderte Gerätezeitzone erst nach
App-Neustart greifen. Workaround falls je nötig:
[@callstack/timezone-hermes-fix](https://github.com/callstack/timezone-hermes-fix)
(Native-Modul, im Dev Client problemlos). Für v1 ist die richtige Antwort:
akzeptieren und dokumentieren — das Gerät steht fest im Kinderzimmer. Damit ist
auch der Karten-Punkt „Reise und Zeitzonenwechsel" entschieden genug, um ihn
aus Ticket 02 herauszuhalten.

**F11 — Niemals ein `Date` aus Wanduhr-Komponenten bauen.** Hermes' Umrechnung
lokal→UTC für mehrdeutige/nicht existierende Zeiten ist eine Schätzung:
`guessUTC = timeMs - ltza_ - MS_PER_HOUR` (`DateCache.cpp` Z. 43–48). In der
Umstellungsnacht ist `new Date(y, m, d, 2, 30)` per Definition mehrdeutig bzw.
nicht existent. Das empfohlene Design liest ausschließlich Komponenten aus
`new Date()` und vergleicht ganze Zahlen — es hat diesen Fehler nicht.

**F12 — Produktseitige Beruhigung:** Mit dem Vorgabeplan (≈19:00 rot / 07:00
grün) liegt an **keiner** der beiden Umstellungen eine Startzeit im betroffenen
Fenster 02:00–03:00. Beide DST-Nächte sind für die Default-Konfiguration
Nicht-Ereignisse; der Aufwand oben schützt gegen selbst gebaute Pläne mit
nächtlichen Abschnitten.

### Testfälle für die reine Zeitfunktion (Deliverable)

Gruppe A braucht **keine** gemockte Uhr (die Funktion sieht nur eine Zahl) —
das ist der eigentliche Gewinn des Zuschnitts. Gemockt/gesetzt wird nur in B/C.

**A — `aktiverAbschnitt(plan, m)`**

| # | Fall | Erwartung |
|---|---|---|
| A1 | Plan mit genau einem Abschnitt (07:00), `m` = 0, 419, 420, 421, 1439 | immer derselbe Abschnitt (24-h-Ring aus einem Element) |
| A2 | `m` exakt auf einer Startzeit | der **beginnende** Abschnitt (Grenze gehört nach vorne) |
| A3 | `m` = Startzeit − 1 | der vorherige Abschnitt |
| A4 | Plan 07:00/19:00, `m` = 0…419 (vor der ersten Startzeit) | der **19:00**-Abschnitt (Wrap über Mitternacht) |
| A5 | Plan 07:00/19:00, `m` = 1140…1439 | der 19:00-Abschnitt |
| A6 | Ein Abschnitt liegt exakt auf 00:00 | bei `m = 0` dieser Abschnitt; Wrap-Logik bricht nicht |
| A7 | Ein Abschnitt liegt auf 23:59 | aktiv bei `m = 1439`; bei `m = 0` weiterhin dieser (Wrap) |
| A8 | Unsortierter Plan als Eingabe | gleiches Ergebnis wie sortiert — oder die Vorbedingung „sortiert" ist explizit getestet/erzwungen |
| A9 | Zwei Abschnitte mit identischer Startzeit | deterministisches Ergebnis gemäß Entscheidung aus Ticket 05 |
| A10 | Leerer Plan | definiertes Verhalten gemäß Ticket 05 (Fallback/`null`), **kein** Throw |
| A11 | Property: alle `m` in 0…1439 | genau ein Abschnitt, nie `undefined` (Totalität) |
| A12 | Zweimal derselbe `m` | identisches Ergebnis (kein versteckter Zustand) |
| A13 | Sweep über 0…1439 | Zahl der Farbwechsel == Zahl der Abschnitte, Reihenfolge == sortierte Startzeiten |
| A14 | **Rücksprung**: Aufruffolge `m = 180`, dann `m = 120` | korrekt der frühere Abschnitt — Funktion darf Monotonie nicht voraussetzen (Herbstumstellung, Uhr zurückgestellt) |
| A15 | **Großer Sprung**: `m = 1380`, dann `m = 420` (8 h Bildschirm aus) | direkt der 07:00-Abschnitt, keine nachgeholten Zwischenschritte |

**B — Adapter `minuteDesTages(date)`** (hier braucht es TZ-Kontrolle im Test-Runner)

| # | Fall | Erwartung |
|---|---|---|
| B1 | 00:00:00.000 / 12:00:00 / 23:59:59.999 | 0 / 720 / 1439 |
| B2 | 06:59:59.999 | 419, nicht 420 (Sekunden und ms werden verworfen) |
| B3 | Derselbe Instant unter `TZ=Europe/Berlin` und `TZ=UTC` | unterschiedliche Werte — beweist, dass lokale Getter benutzt werden |
| B4 | **Frühjahr, Europe/Berlin**: `2027-03-28T00:59:00Z` → lokal 01:59; `2027-03-28T01:00:00Z` → lokal 03:00 | 119 bzw. 180; die Werte 120…179 existieren an diesem Tag nicht |
| B5 | **Herbst, Europe/Berlin**: `2026-10-25T00:30:00Z` (CEST) und `2026-10-25T01:30:00Z` (CET) | **beide** 150 — derselbe Minutenwert zu zwei Instants |
| B6 | Uhr um 5 h vorgestellt (Clock-Mock) | nächster Aufruf liefert sofort den neuen Wert, kein eigenes Caching |

**C — Scheduler-Helfer `msBisNaechstemTick(nowMs)`**

| # | Fall | Erwartung |
|---|---|---|
| C1 | `nowMs` exakt auf einer Minutengrenze | ≈ 60 000, **niemals** 0 |
| C2 | beliebiger Zeitpunkt | Ergebnis > 0 und ≤ 60 000 (+ Epsilon) |
| C3 | Zeitpunkt 10 ms vor der Grenze | ≥ Untergrenze (250 ms) — verhindert Busy-Loop, wenn ein Timer minimal zu früh feuert |
| C4 | verschiedene Zeitzonen | identisches Ergebnis (Minutengrenzen der Epoche == lokale Minutengrenzen, alle Offsets sind ganze Minuten) |
| C5 | Uhr springt zwischen zwei Aufrufen zurück | weiterhin positiver, gedeckelter Wert; kein negativer Timeout, kein Riesenwert |

**D — Tick-/Zustandslogik** (ohne Renderer testbar, gehört zur Zeitlogik dazu)

| # | Fall | Erwartung |
|---|---|---|
| D1 | 60 Ticks innerhalb eines Abschnitts | genau **ein** Zustands-Update (nur bei echtem Wechsel schreiben) |
| D2 | Tickfolge über eine Grenze | genau ein Wechsel, mit dem richtigen Zielabschnitt |
| D3 | `AppState → 'active'` | sofortige Neuberechnung **und** Neuplanung des Timeouts; danach läuft nur ein Timeout, nicht zwei |
| D4 | Unmount | Timeout gelöscht, AppState-Subscription entfernt |
| D5 | Tickfolge nicht monoton (`m` 179 → 120, vgl. A14) | kein Throw, kein „stecken gebliebener" Abschnitt |

### Gerätecheckliste (nicht unit-testbar, gehört in die Bau-Session)

- **E1** Sperrtaste, ≥ 30 min über eine Abschnittsgrenze hinweg warten,
  einschalten: wie lange ist die alte Farbe sichtbar? Auf iOS **und** Android
  messen (Zeitlupenvideo). Das ist die einzige bekannte Stelle, an der die
  Plattform nicht mitspielt, und sie verschiebt evtl. die Produkterwartung.
- **E2** Systemzeit im Vordergrund 10 min über eine Grenze vorstellen (setzt
  voraus, dass man die App verlässt → deckt zugleich den AppState-Pfad ab):
  Farbe korrigiert sich ≤ 1 min.
- **E3** Systemzeit zurückstellen: Farbe geht zurück, nichts hängt.
- **E4** Zeitzone auf UTC umstellen: auf Android **erwarteter Fehlschlag**
  (F10) — prüfen, ob ein Neustart hilft, Ergebnis dokumentieren.
- **E5** Eine echte Nacht durchlaufen lassen; morgens prüfen, ob der
  Android-Prozess noch lebt (sonst Cold Start — dann muss der erste Frame nach
  dem AsyncStorage-Laden korrekt sein, Berührungspunkt zu Ticket 05).

### Nicht verifiziert / offene Unsicherheiten

- `reactnative.dev`, `docs.expo.dev`, `developer.apple.com` und
  `developer.android.com` waren aus dieser Umgebung nicht direkt abrufbar
  (Egress blockiert). Die Aussagen aus diesen Quellen stammen aus
  Suchergebnis-Auszügen; **alle Quellcode-Aussagen** (F1–F3, F4-Code-Teile,
  F5–F7, F9–F11) wurden dagegen in den echten Dateien gelesen — npm-Tarball
  `react-native@0.85.2`, Branch `0.85-stable`, `facebook/hermes@main`,
  `expo/expo@sdk-56`.
- **Android `System.nanoTime()` im Deep Sleep**: nach POSIX-Semantik
  (`CLOCK_MONOTONIC`) läuft es während System-Suspend nicht weiter; eine
  explizite Android-Doku-Stelle dazu habe ich nicht gefunden. Betrifft nur die
  ohnehin verworfene Lange-Timeout-Variante, nicht die Empfehlung.
- **iOS `NSTimer`/`CFRunLoopTimer` bei verstellter Wanduhr**: ob eine bereits
  scharfgestellte Sleep-Timer-Fire-Date neu ausgewertet wird, konnte ich nicht
  belegen. Praktisch entschärft, weil man die Systemzeit nicht ändern kann,
  ohne unsere App zu verlassen → `AppState → 'active'` bricht den Timeout ab
  und plant neu.
- **Hermes-Zeitzonencache auf iOS**: der Hermes-Header behauptet
  Selbstheilung auf Apple-Plattformen (ein veralteter Aufruf, danach Reset),
  `@callstack/timezone-hermes-fix` listet iOS trotzdem als betroffen. Wer von
  beiden für Hermes V1 unter iOS 26 recht hat, ist ungeprüft — messen (E4).
- **Hermes V1 vs. bisherige Hermes-Version**: die gelesenen `DateCache`-Quellen
  liegen auf `facebook/hermes@main`. Dass Hermes V1 in RN 0.85 exakt diesen
  Stand enthält, habe ich nicht Commit-genau nachgewiesen; die
  `resetTimezoneCache`-Deklaration in RNs `hermes-interfaces.h` zeigt aber, dass
  der PR-1693-Stand drin ist.
- **Dauer des Stale-Frames beim Aufwachen** (E1) ist nicht recherchierbar,
  nur messbar. Sollte sie störend lang sein, wäre die einzige mir bekannte
  Gegenmaßnahme eine native (Config-Plugin-)Lösung — dann zurück auf die Karte.
