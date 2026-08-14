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

## Answer

Basis der Recherche: `expo@56.0.19` → **React Native 0.85.3**, `expo-status-bar@56.0.4`,
`expo-navigation-bar@56.0.3`, `expo-keep-awake@56.0.3`, `expo-router@56.2.18`,
`react-native-screens@4.26.2`, `expo-template-bare-minimum@56.0.33`. Alle Aussagen
stammen aus dem Quelltext dieser veröffentlichten Pakete bzw. dem `sdk-56`-Branch
von `expo/expo` — **`docs.expo.dev` und `expo.dev` waren aus dieser Session nicht
erreichbar** (Egress-Proxy blockt beide Domains), siehe „Nicht verifiziert".

---

### Empfehlung — Kurzfassung

**Gemeinsam (beide Plattformen)**

- `app.json` → `plugins`:
  - `["expo-status-bar", { "hidden": true }]`
  - `["expo-navigation-bar", { "hidden": true }]`
- Genau **ein** `<StatusBar hidden />` im Root-Layout (`app/_layout.tsx`) und
  **nirgendwo sonst** eine `StatusBar` ohne `hidden` (siehe Fallstrick unten).
- Auf Android zusätzlich `<NavigationBar hidden />` (neu in SDK 56) im Root-Layout.
- `useKeepAwake()` im Root-Layout. Kein Plugin, keine Permission nötig.
- Farbfläche als schlichtes `<View style={{ flex: 1, backgroundColor }} />`
  **ohne** `SafeAreaView` / Insets — die Fläche soll ja unter die Leisten laufen.
- Beide Plugins wirken über `styles.xml` bzw. `Info.plist` → **Prebuild + neuer
  Dev Build** nötig, nicht per JS-Reload.

**iOS**

- Status Bar: über das `expo-status-bar`-Plugin (`hidden: true` → `UIStatusBarHidden`
  in der Info.plist) **plus** `<StatusBar hidden />` zur Laufzeit. Das Plugin allein
  deckt nur den Start-Frame ab.
- Home Indicator: expo-router-Screen-Option **`autoHideHomeIndicator: true`**
  (`<Stack screenOptions={{ autoHideHomeIndicator: true }} />`). Das geht direkt
  durch bis `prefersHomeIndicatorAutoHidden` in react-native-screens. **Keine
  Zusatzbibliothek nötig** — insbesondere *nicht* `react-native-home-indicator`.
- Ergebnis: Status Bar dauerhaft weg, Home Indicator blendet nach einigen Sekunden
  ohne Berührung aus und kommt bei jeder Berührung zurück. Mehr gibt iOS nicht her.

**Android**

- Status Bar + Navigation Bar über die beiden Config-Plugins (`hidden: true`),
  dadurch schon in `Activity.onCreate()` versteckt — **vor** dem JS-Start, also
  kein sichtbares Aufblitzen der Leisten.
- Zur Laufzeit `<StatusBar hidden />` + `<NavigationBar hidden />` als Absicherung.
- Immersive-Verhalten ist in SDK 56 **nicht mehr konfigurierbar** und fest auf
  `BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE` verdrahtet: Wischen von der Kante zeigt
  die Leisten kurz an, danach verschwinden sie automatisch wieder. Für eine
  Kinderzimmer-Ampel ist genau das das gewünschte Verhalten.
- Edge-to-edge ist erzwungen (siehe unten) — die Farbfläche läuft ohnehin unter
  Status- und Navigationsleiste sowie in den Display-Cutout hinein.

**Nicht empfohlen**

- `statusBarHidden` als expo-router-Screen-Option auf iOS (kollidiert mit dem
  Expo-Default in der Info.plist, s.u.).
- `expo-navigation-bar`-Legacy-API (`setVisibilityAsync`, `useVisibility`) — in
  56 deprecated.
- `react-native-home-indicator` — letzte Veröffentlichung 2022, Paper-ViewManager;
  RN 0.85 hat die Bridge entfernt.

---

### Fakten und Belege

#### 1. Status Bar

**Android (RN 0.85.3).** `StatusBar.setHidden(true)` landet in
`Window.setStatusBarVisibility()`. Bei aktivem edge-to-edge:

```kotlin
WindowInsetsControllerCompat(this, decorView).run {
  systemBarsBehavior = WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
  hide(WindowInsetsCompat.Type.statusBars())
}
```
→ [`WindowUtil.kt`, RN v0.85.3](https://github.com/facebook/react-native/blob/v0.85.3/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/views/view/WindowUtil.kt)

**iOS (RN 0.85.3).** `setHidden` ruft das seit iOS 9 deprecated
`UIApplication.setStatusBarHidden:` und **verweigert die Arbeit mit
`RCTLogError`, wenn `UIViewControllerBasedStatusBarAppearance` auf `YES` steht**:
→ [`RCTStatusBarManager.mm`, RN v0.85.3](https://github.com/facebook/react-native/blob/v0.85.3/packages/react-native/React/CoreModules/RCTStatusBarManager.mm)

Der Expo-Prebuild-Template setzt genau deshalb `UIViewControllerBasedStatusBarAppearance = false`
(geprüft in `ios/HelloWorld/Info.plist` von `expo-template-bare-minimum@56.0.33`).
Damit funktioniert der `expo-status-bar`-Weg — und der react-native-screens-Weg
(`statusBarHidden`) **nicht**, denn dessen Doku sagt ausdrücklich: *„Requires
setting `View controller-based status bar appearance -> YES` in your Info.plist file."*
(`expo-router@56.2.18`, `build/react-navigation/native-stack/types.d.ts`). Man kann
nur eines von beidem haben.

**Config-Plugin (neu in SDK 56).** `expo-status-bar` hat jetzt ein eigenes Plugin:
`hidden: true` schreibt `UIStatusBarHidden` in die Info.plist und
`expoStatusBarHidden` als Theme-Attribut in `styles.xml`; ein
`ReactActivityLifecycleListener` versteckt die Leiste damit auf Android schon in
`onCreate`, „before the JS engine starts".
→ [`withStatusBar.ts`](https://github.com/expo/expo/blob/sdk-56/packages/expo-status-bar/plugin/src/withStatusBar.ts),
[`StatusBarReactActivityLifecycleListener.kt`](https://github.com/expo/expo/blob/sdk-56/packages/expo-status-bar/android/src/main/java/expo/modules/statusbar/StatusBarReactActivityLifecycleListener.kt),
[CHANGELOG 56.0.0](https://github.com/expo/expo/blob/sdk-56/packages/expo-status-bar/CHANGELOG.md)

> **Fallstrick, der uns sonst beißt:** RNs `StatusBar` mergt einen Props-Stack und
> fällt auf `_defaultProps.hidden = false` zurück, sobald *irgendeine* gemountete
> `StatusBar` kein `hidden` gesetzt hat. Ein übrig gebliebenes
> `<StatusBar style="auto" />` aus dem Expo-Template holt die Status Bar also
> aktiv wieder zurück — trotz `UIStatusBarHidden` in der Info.plist. Genau **eine**
> `<StatusBar hidden />` im Root, sonst keine.
> → [`StatusBar.js`, RN v0.85.3](https://github.com/facebook/react-native/blob/v0.85.3/packages/react-native/Libraries/Components/StatusBar/StatusBar.js)

Apple: `UIStatusBarHidden` ist definiert als *„whether the system initially hides
the status bar when the app launches"* — also nur der Startzustand, kein Riegel.
→ [Apple, UIStatusBarHidden](https://developer.apple.com/documentation/bundleresources/information-property-list/uistatusbarhidden)

#### 2. Android Navigation Bar / Immersive Mode

SDK 56 ist ein Bruch. Aus dem [CHANGELOG von `expo-navigation-bar` 56.0.0](https://github.com/expo/expo/blob/sdk-56/packages/expo-navigation-bar/CHANGELOG.md):

- **Entfernt**: `setBehaviorAsync`/`getBehaviorAsync`, `setPositionAsync`,
  `setBackgroundColorAsync`, `setBorderColorAsync`, `setButtonStyleAsync` sowie die
  Plugin-Properties `backgroundColor`, `borderColor`, `behavior`, `position`.
- **Neu**: `NavigationBar`-Komponente mit `style`/`hidden` und
  `NavigationBar.setHidden` (Stack-Merging wie bei `StatusBar`).
- **Deprecated**: `setVisibilityAsync`, `getVisibilityAsync`, `useVisibility`,
  `addVisibilityListener`, Top-Level-`setStyle`.
- Plugin-Properties heißen jetzt `style`/`hidden` statt `barStyle`/`visibility`.
- Das alte `androidNavigationBar` in `app.json` ist in SDK 56 **komplett aus den
  Expo-Config-Typen verschwunden** (in `@expo/config-types@54` noch vorhanden, in
  55 und 56 nicht mehr). `androidStatusBar` existiert noch, ist aber als
  *„@deprecated Use the `expo-status-bar` plugin configuration instead"* markiert.

Der aktuelle Weg, im Quelltext:

```kotlin
AsyncFunction("setHidden") { hidden: Boolean ->
  WindowInsetsControllerCompat(window, window.decorView).run {
    systemBarsBehavior = WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
    when (hidden) { true -> hide(...navigationBars()); false -> show(...) }
  }
}
```
→ [`NavigationBarModule.kt`](https://github.com/expo/expo/blob/sdk-56/packages/expo-navigation-bar/android/src/main/java/expo/modules/navigationbar/NavigationBarModule.kt)

Das Verhalten ist damit **fest verdrahtet und nicht mehr wählbar**. Android
dokumentiert es so: *„Use `BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE` to temporarily
reveal hidden system bars with system gestures … These transient system bars
overlay your app's content, might have some degree of transparency, and are
automatically hidden after a short timeout."*
→ [Android: Hide system bars for immersive mode](https://developer.android.com/develop/ui/views/layout/immersive)

Das ist die Antwort auf „wie zuverlässig hält er, wenn der Nutzer an den Rand
wischt": **Er hält.** Die Leiste taucht beim Kantenwisch kurz auf und verschwindet
nach einem kurzen Timeout von selbst wieder. Kein Zustand, den man zurücksetzen muss.

Das Plugin schreibt `expoNavigationBarHidden` als Theme-Attribut, ein
`ReactActivityLifecycleListener` versteckt die Leiste in `onCreate`:
→ [`withNavigationBar.ts`](https://github.com/expo/expo/blob/sdk-56/packages/expo-navigation-bar/plugin/src/withNavigationBar.ts),
[`NavigationBarReactActivityLifecycleListener.kt`](https://github.com/expo/expo/blob/sdk-56/packages/expo-navigation-bar/android/src/main/java/expo/modules/navigationbar/NavigationBarReactActivityLifecycleListener.kt)

Der alte Bug „Leiste kommt bei der ersten Berührung zurück und bleibt"
([expo/expo#6525](https://github.com/expo/expo/issues/6525)) stammt aus der Ära der
`SYSTEM_UI_FLAG_*`-Flags und ist mit `WindowInsetsController` konstruktiv erledigt.

#### 3. Edge-to-edge auf Android

Ja, erzwungen — und zwar doppelt:

- **Expo-seitig**: Der Schalter ist weg. `edgeToEdgeEnabled` existiert in
  `@expo/config-types@54`, aber nicht mehr in `@expo/config-types@55/56`, und auch
  nicht in `expo-build-properties@56`. Im Prebuild-Template steht
  `edgeToEdgeEnabled=true` fest in `android/gradle.properties`.
  Aus dem `expo-navigation-bar`-CHANGELOG 55.x: die Farb-/Behavior-APIs wurden
  no-op'd, *„now that edge-to-edge is mandatory on Android"*.
- **Plattform-seitig**: Expo 56 baut mit `targetSdkVersion 36`
  (`ExpoModulesCorePlugin.gradle` in `expo-modules-core@56.0.23`). Android dazu:
  *„For apps targeting Android 16 (API level 36), `R.attr#windowOptOutEdgeToEdgeEnforcement`
  is deprecated and disabled, and your app can't opt-out of going edge-to-edge."*
  → [Android 16 behavior changes](https://developer.android.com/about/versions/16/behavior-changes-16)

Für unsere Fläche ist das ein Geschenk: RNs `enableEdgeToEdge()` setzt
`setDecorFitsSystemWindows(false)`, transparente Systemleisten und ab API 30
`LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS` — die Farbe läuft also bis in die
Notch/Punch-Hole hinein. Konsequenz für uns: **keine Safe Areas auf dem
Home-Screen**. Die Expo-Doku warnt umgekehrt, dass man ab jetzt Safe Areas
*braucht*, wenn Inhalt nicht unter die Leisten soll — für den Settings-Screen mit
Liste und Eingaben also durchaus relevant.
→ [`docs/pages/develop/user-interface/system-bars.mdx` @ sdk-56](https://github.com/expo/expo/blob/sdk-56/docs/pages/develop/user-interface/system-bars.mdx)

#### 4. iOS Home Indicator

**Es gibt keinen „aus"-Schalter, nur „auto-hide".** Die Property heißt
`prefersHomeIndicator**AutoHidden**`, und genau das tut sie: der Indikator blendet
nach ein paar Sekunden ohne Berührung aus und **kommt bei jeder Berührung zurück**.
Apple formuliert es zusätzlich als Wunsch, nicht als Befehl: das System
berücksichtigt die Präferenz, garantiert das Ausblenden aber nicht.
→ [Apple: prefersHomeIndicatorAutoHidden](https://developer.apple.com/documentation/uikit/uiviewcontroller/prefershomeindicatorautohidden),
[Hacking with Swift: How to hide the home indicator](https://www.hackingwithswift.com/example-code/uikit/how-to-hide-the-home-indicator-on-iphone-x)

**Der Weg dahin ohne Fremdbibliothek** ist in Expo 56 fertig verdrahtet:

- `expo-router@56.2.18` bringt einen vendorierten native-stack mit der Option
  `autoHideHomeIndicator?: boolean` (*„Whether the home indicator should prefer to
  stay hidden on this screen. Defaults to `false`. @platform ios"*), die als
  `homeIndicatorHidden` an `react-native-screens` durchgereicht wird
  (`build/react-navigation/native-stack/views/NativeStackView.native.js`).
  `expo-router`s `Stack`-`screenOptions` sind vom Typ `NativeStackNavigationOptions`,
  die Option ist also direkt verfügbar.
- `react-native-screens@4.26.2` implementiert daraus
  `- (BOOL)prefersHomeIndicatorAutoHidden { return self.screenView.homeIndicatorHidden; }`
  und **swizzelt** `childViewControllerForHomeIndicatorAutoHidden` auf
  `UIViewController`, damit die Präferenz vom Root-VC bis zum Screen durchpropagiert.
  → [`ios/RNSScreen.mm`](https://github.com/software-mansion/react-native-screens/blob/4.26.2/ios/RNSScreen.mm),
  [`ios/UIViewController+RNScreens.mm`](https://github.com/software-mansion/react-native-screens/blob/4.26.2/ios/UIViewController%2BRNScreens.mm)

**Nicht nehmen:** `react-native-home-indicator` — letzte Version 0.2.10 vom
20.09.2022, klassischer Paper-ViewManager.
[RN 0.85 (April 2026) hat die Bridge aus dem Code entfernt](https://reactnative.dev/blog/2026/04/07/react-native-0.85);
seit RN 0.82 / Expo SDK 55 gibt es die Legacy-Architektur nicht mehr. Das Paket ist
mit Expo 56 mit hoher Wahrscheinlichkeit unbrauchbar.
→ [npm: react-native-home-indicator](https://www.npmjs.com/package/react-native-home-indicator)

#### 5. Sleep verhindern

`expo-keep-awake` ist der richtige und einzige nötige Weg. Kein Config-Plugin, kein
Manifest-Eintrag, keine Permission (`AndroidManifest.xml` des Pakets ist leer).

- **Android**: `activity.window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)`
  → [`ExpoKeepAwakeManager.kt`](https://github.com/expo/expo/blob/sdk-56/packages/expo-keep-awake/android/src/main/java/expo/modules/keepawake/ExpoKeepAwakeManager.kt)
- **iOS**: `UIApplication.shared.isIdleTimerDisabled = true`, plus `OnAppEntersForeground`/
  `OnAppEntersBackground`, die das Flag beim Backgrounding lösen und beim
  Zurückkehren wieder setzen.
  → [`ios/KeepAwakeModule.swift`](https://github.com/expo/expo/blob/sdk-56/packages/expo-keep-awake/ios/KeepAwakeModule.swift)

**Beides sind Zustands-Flags ohne Timeout.** Zur Frage „gilt das auch, wenn die App
stundenlang unberührt im Vordergrund steht": ja — es gibt keinen Zeitzähler, der
irgendwann abläuft. Solange die App im Vordergrund ist und die Komponente mit
`useKeepAwake()` gemountet bleibt, schläft der Bildschirm nicht ein. Genau das
Szenario der Karte (Gerät hängt am Strom, läuft die ganze Nacht) ist abgedeckt.

`useKeepAwake()` vergibt per `useId()` automatisch ein eindeutiges Tag und
deaktiviert beim Unmount — einmal im Root-Layout, fertig.

#### 6. Was braucht nativen Build?

| Punkt | Mechanismus | Nur im Dev Build? |
|---|---|---|
| `expo-status-bar` Plugin `hidden` | Info.plist `UIStatusBarHidden`, `styles.xml` `expoStatusBarHidden` | ja — Prebuild + Rebuild |
| `expo-navigation-bar` Plugin `hidden` | `styles.xml` `expoNavigationBarHidden` | ja — Prebuild + Rebuild |
| `<StatusBar hidden />` / `<NavigationBar hidden />` | Laufzeit-Module | nein, JS-Reload reicht |
| `autoHideHomeIndicator` | Laufzeit-Prop auf react-native-screens | nein, JS-Reload reicht |
| `useKeepAwake()` | Laufzeit-Modul | nein |
| edge-to-edge | Gradle/targetSdk, nicht abschaltbar | n/a |

Da ohnehin nur Dev Builds per Kabel ausgeliefert werden (Kartenentscheidung), ist
das für uns kein Kostenpunkt — es bedeutet nur: **die beiden Plugins müssen vor dem
ersten Dev Build in `app.json` stehen**, sonst kostet es eine zweite Build-Runde.

---

### Was die Plattform *nicht* zulässt

Das ist der Teil, der die Produkterwartung verschiebt.

**iOS**

1. **Der Home Indicator lässt sich nicht dauerhaft entfernen.** `prefersHomeIndicatorAutoHidden`
   blendet ihn nach Inaktivität aus; **jede Berührung holt ihn zurück**. Da das Kind
   die App per Long-Press bedient (Weg in die Settings), wird er regelmäßig sichtbar.
   Es gibt in UIKit keinen „hidden"-Schalter, nur „autoHidden".
2. **Der Wisch-nach-oben lässt sich nicht abschalten.** Es gäbe
   `preferredScreenEdgesDeferringSystemGestures` (erster Wisch wird verschluckt,
   zweiter greift) — aber `react-native-screens@4.26.2` implementiert das **nicht**
   (im gesamten `ios/`-Verzeichnis kein einziges Vorkommen). Ohne eigenes
   Native-Modul / eigenes Config-Plugin ist das nicht erreichbar. Und selbst damit
   wäre es nur eine Verzögerung, keine Sperre.
3. **Notification Center und Control Center** sind vom Rand aus immer erreichbar und
   legen sich über die Farbfläche. Nicht verhinderbar.
4. **`UIApplication.setStatusBarHidden:` ist deprecated** (seit iOS 9) und der einzige
   Weg, den RN 0.85 für die Status Bar anbietet. Funktioniert heute, ist aber
   erklärtermaßen Altlast — bei einem künftigen iOS-Release ein realistischer
   Bruchkandidat.
5. **Status Bar und Home Indicator schließen sich in der Konfiguration aus**, wenn man
   den react-native-screens-Weg für beides will: `statusBarHidden` (RNS) braucht
   `UIViewControllerBasedStatusBarAppearance = YES`, RNs/`expo-status-bar`s
   `setHidden` braucht `NO`. Man muss sich entscheiden. Expo-Default ist `NO`.
6. **Der einzige echte Vollbild-Riegel ist „Geführter Zugriff"** (Guided Access,
   dreimal Seitentaste), vom Nutzer am Gerät zu aktivieren, nicht von der App. Dort
   ist der Home-Indikator tatsächlich unsichtbar und die Systemgesten sind
   blockiert. Das ist eine Betriebsanleitung fürs Gerät, kein Feature der App.
   → [Macworld: How to remove the home bar](https://www.macworld.com/article/675061/how-to-remove-the-home-bar-at-bottom-of-iphone-screen.html)

**Android**

7. **Die Systemleisten kommen beim Kantenwisch immer kurz zurück.** Nicht
   abschaltbar — und in SDK 56 nicht einmal mehr konfigurierbar, seit
   `setBehaviorAsync` entfernt wurde. Immerhin verschwinden sie von selbst wieder.
8. **Edge-to-edge kann nicht abgeschaltet werden**, weder über Expo (Option aus den
   Config-Typen entfernt) noch über die Plattform (targetSdk 36 → Android 16
   ignoriert `windowOptOutEdgeToEdgeEnforcement`).
9. **Kein echter Kiosk-Modus ohne Fremdbestimmung.** „App-Anheftung"/Screen Pinning
   muss der Nutzer am Gerät einschalten; `startLockTask()` (Lock Task Mode) setzt
   einen Device Owner via MDM/ADB voraus. Beides liegt außerhalb dessen, was ein Dev
   Build per Kabel mitbringt.
10. **Benachrichtigungs-Heads-Up-Banner** legen sich über die Farbfläche. Nicht
    verhinderbar (Lösung liegt beim Gerät: „Nicht stören").
11. **`FLAG_KEEP_SCREEN_ON` schützt nicht vor der Sperrtaste** — was hier ausdrücklich
    gewollt ist (Karte: „Bildschirm wird manuell per Sperrtaste ausgeschaltet"),
    aber es heißt eben auch: die App kann den Bildschirm nicht wieder anschalten.

**Beide**

12. **Keep-Awake endet mit dem Backgrounding.** Sobald die App nicht mehr im
    Vordergrund ist, greift wieder das normale Display-Timeout. Für „App liegt nachts
    im Vordergrund" irrelevant, für „App ist versehentlich im Hintergrund" nicht.
13. **Ein Vollbild-Farbfeld über Stunden ist OLED-Einbrennrisiko.** Keine
    Plattformgrenze, aber eine reale Nebenwirkung des Produkts — ggf. Notiz für die
    Spec.

---

### Nicht verifiziert (bewusst offen gelassen)

- **`docs.expo.dev` und `expo.dev` waren aus dieser Session nicht abrufbar** (Egress-Proxy
  blockt die Domains). Alle Expo-Aussagen stammen ersatzweise aus dem `sdk-56`-Branch
  von `expo/expo` und aus den Tarballs der veröffentlichten 56.x-npm-Pakete. Die
  offiziellen Doku-Seiten zu `navigation-bar`, `status-bar` und dem Edge-to-Edge-Blogpost
  sollte jemand mit Netzzugang gegenprüfen.
- **Nichts davon wurde auf einem Gerät oder Emulator getestet** — das Ticket ist
  Research, es wurde kein Projekt angelegt und nichts installiert.
- **Ob die versteckten Leisten einen Screen-Off/Screen-On-Zyklus auf Android überleben,
  ist offen.** `expo-navigation-bar` versteckt via Plugin nur in `Activity.onCreate`;
  `react-native-screens` reapplied die Window-Traits in `ScreenFragment.onResume()`
  nur bedingt (`shouldUpdateOnResume`, gesetzt wenn beim Anwenden keine Activity da
  war). Empfehlung als billige Versicherung: beim `AppState`-Wechsel auf `'active'`
  einmal `StatusBar.setHidden(true)` und `NavigationBar.setHidden(true)` nachschieben —
  und das auf dem Zielgerät verifizieren.
- **Ob die Software-Tastatur auf dem Settings-Screen die Navigationsleiste dauerhaft
  zurückholt**, ist ungeprüft. Betrifft nur den Settings-Screen, aber ggf. muss nach
  dem Schließen der Tastatur nachgefasst werden.
- **Die genaue Ausblendzeit des iOS Home Indicators** („ein paar Sekunden", ~3 s) ist
  von Apple nicht dokumentiert; die Angabe stammt aus Sekundärquellen.
- **Guided Access / Home-Indikator** ist über Sekundärquellen (Macworld, Kiosk-Anbieter-KB)
  belegt, nicht über Apples eigene Doku.
- **`react-native-home-indicator` ist unter Expo 56 nicht getestet**, sondern nur aus
  Veröffentlichungsdatum (2022) und dem Bridge-Removal in RN 0.85 als untauglich
  eingeschätzt. Da wir es nicht brauchen, wurde das nicht weiter verfolgt.
- **`expo-router@56` hängt nicht mehr an `@react-navigation/*`**, sondern bringt einen
  vendorierten Stack plus ein Paket `standard-navigation@^0.0.5` mit. Die hier
  genutzten Optionen (`autoHideHomeIndicator`, `statusBarHidden`, `navigationBarHidden`)
  stehen belegbar in `expo-router/build/react-navigation/native-stack/types.d.ts` und
  werden nachweislich an `ScreenStackItem` durchgereicht — die weiteren Folgen dieses
  Umbaus wurden nicht untersucht.
