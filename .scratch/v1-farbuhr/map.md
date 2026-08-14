# Wayfinder Map: v1 Farbuhr

Label: `wayfinder:map`

## Destination

Eine implementierungsreife Spec für die v1 der Expo-App: eine Farbfläche im
Vollbild, deren Farbe der Systemuhr folgt, plus ein Settings-Screen zum
Bearbeiten des Plans. Die Karte ist geräumt, wenn nichts mehr zu entscheiden
ist, bevor jemand die App baut — gebaut wird in eigenen Sessions danach.

## Notes

- **Domain**: Kinderzimmer-Aufsteh-Ampel. Gerät hängt dauerhaft am Strom, läuft
  die ganze Nacht, Bildschirm wird manuell per Sperrtaste ausgeschaltet.
  Systemhelligkeit, kein Dimmen durch die App.
- **Diese Karte plant, sie baut nicht.** Kein Produktionscode in
  Wayfinder-Sessions. Prototypen sind explizit Wegwerf-Artefakte zum Anschauen.
- **Glossar**: [CONTEXT.md](../../CONTEXT.md) — insbesondere *Abschnitt*,
  *Plan*, *Aktiver Abschnitt*. Der Begriff „Zeitpunkt" ist bewusst ersetzt.
- **Skills jede Session**: `/grilling` und `/domain-modeling`. Prototyp-Tickets
  zusätzlich `/prototype`, Research-Tickets laufen als `/research` Subagent.
- **Gesetzte Technik** (v1, nicht mehr zu verhandeln): Expo 56, Dev Client (kein
  Expo Go), expo-router mit zwei Screens (Home + Settings), AsyncStorage,
  iOS und Android, Auslieferung nur als Dev Build per Kabel.
- **Zeitlogik ist eine reine Funktion** und wird mit Unit-Tests plus gemockter
  Uhr abgedeckt. Das ist eine Anforderung, keine Option.
- **Der Expo-56-Unterbau** (aus `bundledNativeModules.json` in `expo@56.0.19`,
  publiziert 2026-08-06): React Native 0.85.3, Reanimated 4.3.1,
  Gesture Handler ~2.31.1, Worklets 0.8.3, AsyncStorage 2.2.0,
  `@react-native-community/slider` 5.2.0. Reanimated 4 heißt **New
  Architecture only** — Libraries, die daran nicht angepasst sind, fallen
  raus. Reanimated und Gesture Handler liegen bereits im Default-Template.
  *(ermittelt in Ticket 03)*
- **Vorbehalt zur Recherche**: Die Research-Tickets liefen hinter einem
  Egress-Proxy, der `docs.expo.dev` und `github.com` blockiert. Die Ergebnisse
  stützen sich auf npm-Tarballs und `raw.githubusercontent.com` — also auf
  ausgelieferten Code, nicht auf Dokumentation. **Issue-Tracker konnten nicht
  eingesehen werden**, offene Bugs sind daher nur dort erfasst, wo sie zufällig
  auffindbar waren. Nichts wurde auf einem Gerät ausgeführt. Jede
  Ticket-Antwort hat einen Abschnitt „Nicht verifiziert" — vor der
  Implementierung lesen.

## Decisions so far

<!-- one line per closed ticket: gist + link -->

- **Ziel der Karte** — Ergebnis ist eine Spec zum Übergeben, nicht die gebaute
  App. Implementierung folgt in eigenen Sessions. *(Grilling Runde 1)*
- **Plattform & Auslieferung** — iOS und Android, Dev Client statt Expo Go, nur
  Dev Builds per Kabel auf eigene Geräte. *(Grilling Runde 1)*
- **Zweck** — Farben fürs Kinderzimmer nachts, Gerät dauerhaft an,
  Systemhelligkeit, Bildschirm manuell per Sperrtaste aus. Kein Dimmen durch die
  App. *(Grilling Runde 1)*
- **Ein Plan für alle Tage** — keine Wochentag/Wochenende-Unterscheidung, keine
  Profile. Bewusst simpel gehalten. *(Grilling Runde 1)*
- **Ring-Semantik** — ein Abschnitt ist der Beginn einer Strecke, nicht ein
  Ereignis; die Liste ist ein 24-Stunden-Ring, der über Mitternacht
  zurückwickelt. Begriff „Zeitpunkt" durch **Abschnitt** ersetzt.
  *(Grilling Runde 2 → [CONTEXT.md](../../CONTEXT.md))*
- **Erststart** — leere Liste, **keine** Vorgabe-Abschnitte; der Settings-Screen
  ist das Onboarding, weil er mit 0 Abschnitten nicht verlassen werden kann.
  *(Grilling Runde 2, ersetzt in
  [05 — Datenmodell des Plans](issues/05-datenmodell-und-kantenfaelle.md) —
  die frühere Vorgabe 19:00 rot / 07:00 grün ist aufgehoben)*
- **Keine Kindersicherung** — Long-Press in Standardlänge ist die einzige Tür zu
  den Settings, kein PIN und keine verlängerte Geste. *(Grilling Runde 2)*
- **Testbarkeit** — Unit-Tests mit gemockter Uhr, kein Debug-Zeitraffer im
  Produkt. *(Grilling Runde 2)*
- **[03 — Welche Farbwahl-Library?](issues/03-farbwahl-library.md)** — gar keine.
  Eine selbstgebaute Palette fester Farbfelder gewinnt; ein HSV-Wähler
  optimiert genau die Gegenrichtung zu dem, was hier gebraucht wird, und trifft
  dunkle Nachttöne am schlechtesten. Rückfallweg wäre
  `reanimated-color-picker` 5.1.2. Gespeichert wird so oder so ein Hex-String,
  ein späterer Wechsel bricht das Datenmodell also nicht.
- **[01 — Vollbild ohne System-Chrome und Wachhalten](issues/01-vollbild-und-wachhalten.md)**
  — machbar ohne Fremd-Libraries, aber SDK 56 hat die API-Fläche umgebaut:
  `<NavigationBar hidden />` statt der entfernten `setBehaviorAsync`-Familie,
  `expo-status-bar` mit Config-Plugin, `androidNavigationBar` aus der
  Expo-Config verschwunden. Android-Immersive ist fest auf
  „Wischen zeigt die Leisten kurz, dann verstecken sie sich wieder"
  verdrahtet und nicht mehr konfigurierbar; Edge-to-edge ist doppelt
  erzwungen. Home Indicator über `autoHideHomeIndicator` in expo-router,
  keine Fremd-Library nötig. Wachhalten ist trivial und hält unbegrenzt.
  **Zwei Fallen vor dem ersten Build**: ein übriggebliebenes
  `<StatusBar style="auto" />` aus dem Template macht das Verstecken still
  wieder rückgängig, und die zwei Wege zum Verstecken der Status Bar
  verlangen gegensätzliche Werte für
  `UIViewControllerBasedStatusBarAppearance` — man muss sich für einen
  entscheiden.
- **[02 — Wann merkt die App den Abschnittswechsel?](issues/02-zeitticking-und-lifecycle.md)**
  — **pollen und zustandslos neu rechnen, nie ein Timeout auf die Startzeit.**
  Die reine Funktion nimmt eine Minute des Tages (`0..1439`) statt eines
  `Date` und kennt weder Zeitzone noch Sommerzeit; ein kurzer,
  selbst-neu-geplanter Timeout auf die nächste volle Minute rechnet jedes Mal
  komplett neu, `AppState → active` rechnet sofort neu. Der exakte Timeout
  scheidet aus, weil iOS Timer gegen die Wanduhr und Android gegen eine
  monotone Uhr rechnet — derselbe Code wäre auf beiden Plattformen
  *unterschiedlich* falsch, und an der Frühjahrsumstellung um exakt eine
  Stunde. Bildschirm aus hält die JS-Timer auf beiden Plattformen an, beim
  Einschalten feuert einmal nach (kein Nachhol-Sturm). Ein Event für
  Uhr- oder Zeitzonensprünge existiert in RN nicht. Liefert außerdem
  **31 konkrete Testfälle** plus eine Geräte-Checkliste.
- **[04 — Welche Zeitwahl-Library?](issues/04-zeitwahl-library.md)** —
  `@react-native-community/datetimepicker` 9.1.0 (von Expo 56 gepinnt), immer
  `mode="time"`: auf Android imperativ als nativer Dialog, auf iOS als
  eingebettete Zeit-Kachel. Zweite Wahl `@expo/ui/community/datetime-picker`,
  falls Ticket 06 einen **inline** eingebetteten Wähler auf Android oder
  Material-3-Optik will — der Umstieg ist ein Import und ein paar Prop-Namen,
  die Entscheidung darf bis nach dem Prototyp warten. Wichtig für Ticket 05:
  der Wähler liefert technisch ein volles `Date`, gespeichert werden trotzdem
  nur Stunde und Minute.
- **[05 — Datenmodell des Plans und seine Kantenfälle](issues/05-datenmodell-und-kantenfaelle.md)**
  — ein Abschnitt ist `{ startMinute, name, color }`, **seine Startzeit ist
  seine Identität** (keine ID): Speichern auf eine belegte Startzeit ersetzt den
  bestehenden Abschnitt still, doppelte Startzeiten sind damit strukturell
  unmöglich. `startMinute` ist `0…1439` — genau die Eingabe der reinen Funktion.
  Der **leere Plan ist legal** und wird nicht verboten, sondern von einem Gate
  abgefangen: mit 0 Abschnitten kann man die Settings nicht verlassen. Ein
  einzelner Abschnitt bleibt zulässig (24 h eine Farbe), eine Obergrenze gibt es
  nicht. Sortierung ist abgeleitet (aufsteigend ab 00:00), nie gespeichert.
  Persistenz: ein Schlüssel `kids-clock/plan`, ein Dokument
  `{ version: 1, segments: [...] }`, Version im Dokument statt im Schlüssel;
  **jeder** Lesefehler verwirft das ganze Dokument und startet leer — billig,
  weil der leere Plan legal ist. Code und Speicher sind **englisch**
  (`Segment`, `Plan`, `activeSegment`), Prosa bleibt deutsch. Ticket 02 erbt
  daraus eine neue Testgruppe für `parsePlan`; A9 wandert von der Logik in die
  Validierung.

## Not yet specified

<!-- in-scope fog: real, but not yet sharp enough to ticket -->

- **Status-Bar-Pfad festlegen**: Die zwei Wege zum Verstecken der iOS-Status-Bar
  verlangen gegensätzliche Werte für `UIViewControllerBasedStatusBarAppearance`,
  und Expos Prebuild-Template liefert einen davon. Eine kleine, aber echte
  Entscheidung — wird beim Schreiben der Spec scharf. *(aus Ticket 01)*
- **App-Identität**: Name, Bundle-ID / Package-Name, Icon, Splash Screen. Klar
  nötig für einen Dev Build, aber es hängt nichts daran, solange die Screens
  nicht stehen.
- **Dev-Build-Erstellung und Kabel-Installation** auf beiden Plattformen:
  Signing, Provisioning, die konkreten Schritte. Wird erst scharf, wenn
  feststeht, welche Native-Module (Ticket 01, 03, 04) überhaupt drin sind.
- **Repo-Grundgerüst**: TypeScript-Konfiguration, Linting, Wahl des Test-Runners.
  Fällt vermutlich beim Schreiben der Spec nebenbei ab.
- **Zeitzonenwechsel als Restrisiko**: Ticket 02 hat den Fall geklärt — es gibt
  kein Event, und wegen eines Caches in der JS-Engine greift ein Zeitzonen-
  wechsel unter Umständen erst nach App-Neustart. Offen ist nur noch, ob das
  für eine Uhr, die dauerhaft im selben Kinderzimmer steht, überhaupt
  erwähnenswert ist oder stillschweigend akzeptiert wird.

## Out of scope

<!-- ruled beyond the destination; never graduates -->

- **Wochentag/Wochenende-Pläne, mehrere Kinder oder Profile** — bewusst gegen
  Komplexität entschieden, ein Plan gilt für alle Tage.
- **Töne, Alarme, Übergangs-Animationen zwischen Farben, Cloud-Sync,
  PIN-Schutz** — alles v2 oder nie.
- **Store-Veröffentlichung** (App Store / Play Store) — eine ganz andere
  Effort-Größe. Auslieferung ist Dev Build per Kabel.
- **Expo Go als Zielumgebung** — Dev Client ist gesetzt, damit Native-Module
  offen stehen.
- **Helligkeitssteuerung oder Dimmen durch die App** — die App nutzt die
  Systemhelligkeit. Ein dunkler Farbton löst den Nacht-Fall.
- **Debug-Modus mit simulierter Uhrzeit** — Unit-Tests mit Mocks decken das ab.
- **Kindersicherung über den Long-Press hinaus** — kein PIN, keine
  Spezialgeste.
