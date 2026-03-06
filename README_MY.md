# MeteoStation — Deutsch

Wetterstation auf Basis der **ESP32‑C3** mit den Sensoren **AHT20** + **BMP280** und einer Android‑App auf Jetpack Compose.

Die Station unterstützt zwei Betriebsarten:

- **BLE (Bluetooth Low Energy)** — Ersteinrichtung und Übertragung der Wetterdaten über BLE.
- **Wi‑Fi STA (Client)** — das Board verbindet sich mit dem Router und liefert Daten per HTTP, inkl. Webseite und JSON‑Metriken.

Die Android‑App verbindet sich mit der Station, zeigt Messwerte an, erlaubt den Moduswechsel und konfiguriert Wi‑Fi **ohne Firmware‑Anpassung**.

---

## Repository‑Struktur

- **`firmware/`** — Firmware für ESP32‑C3 (Arduino / ESP32‑Plattform):
  - Zwei Betriebsarten: **BLE** und **Wi‑Fi STA**.
  - Auto‑Erkennung der I2C‑Sensoren (AHT20, BMP280).
  - Wetterdaten als JSON:
    - über BLE (Characteristic Notifications);
    - über HTTP (`/metrics`) im Wi‑Fi‑Modus.
  - Eingebauter Webserver mit einfachem HTML/JS‑UI (`/`).
  - Service‑JSON‑Nachrichten über BLE zu Moduswechsel und Wi‑Fi‑Status.

- **`android_app/`** — Android‑App (Kotlin + Jetpack Compose):
  - Interaktion mit der Station über BLE und HTTP.
  - Automatische Aktualisierung des Modus‑Icons (BLE / Wi‑Fi STA).
  - Anzeige von Temperatur, Luftfeuchte, Druck, Höhe.
  - WLAN‑Scan, SSID‑Wahl und Passworteingabe.
  - Senden der WLAN‑Einstellungen an das Board per BLE, ohne Flashen.
  - Optimiertes **Release‑APK** (R8 + shrinkResources, minimale Logs).

---

## Flash der ESP32‑C3

### Quellen

Hauptsketch:

```text
firmware/meteo_station/meteo_station.ino
```

In Arduino IDE oder PlatformIO öffnen.

### Benötigte Bibliotheken

- `NimBLE-Arduino`
- `Adafruit_AHTX0`
- `Adafruit_BMP280`

### Wi‑Fi‑Konfiguration

SSID/Passwort wurden früher als Konstanten gesetzt (`WIFI_STA_SSID`, `WIFI_STA_PASSWORD`). Jetzt wird **Wi‑Fi aus der Android‑App per BLE konfiguriert**:

- Board startet im BLE‑Modus.
- App verbindet sich per BLE.
- Nutzer wählt WLAN und gibt Passwort ein.
- App sendet JSON mit Einstellungen an die ESP32.
- ESP32 schaltet auf Wi‑Fi STA, verbindet sich mit dem Router und sendet Status zurück.

Firmware muss nicht mehr für jeden Nutzer angepasst werden.

### Build und Flash

1. ESP32‑Board‑Support in Arduino IDE installieren.
2. `firmware/meteo_station/meteo_station.ino` öffnen.
3. Board **LOLIN C3 Mini** / **ESP32‑C3** wählen.
4. Kompilieren und auf das Board laden.

Nach erfolgreicher WLAN‑Verbindung erhält die Station eine IP. Unter dieser IP erreichbar:

- Web‑Seite: `http://<ip_adresse>/`
- JSON‑Metriken: `http://<ip_adresse>/metrics`

Zusätzlich sendet die ESP32 per BLE Service‑JSON‑Nachrichten:

- `{"type":"wifi_status","status":"ok","ip":"..."}` — WLAN verbunden.
- `{"type":"wifi_status","status":"error","reason":"..."}` — Fehler.
- `{"type":"mode_changed","mode":"BLE"|"STA"}` — Moduswechsel.

Überflüssige Serial‑Logs sind per Flag `kDebugSerialOutput = false` deaktiviert; es bleiben nur Wetter‑JSON und Antworten.

---

## Android‑App bauen

### Anforderungen

- Android Studio mit Kotlin & Jetpack Compose.
- JDK 17.

### Haupt‑Technologien

- Kotlin, Coroutines, StateFlow.
- Jetpack Compose, Material3.
- OkHttp für HTTP‑Requests.
- `kotlinx.serialization` für JSON.

Versionen sind in `android_app/app/build.gradle.kts` hinterlegt (Kotlin 1.9.x, Compose BOM `2023.10.01`).

### Debug‑Build

```bash
cd android_app
./gradlew assembleDebug
```

APK liegt unter:

```text
android_app/app/build/outputs/apk/debug/app-debug.apk
```

### Release‑Build (optimiertes APK)

In `android_app/app/build.gradle.kts` ist das Release‑Profil konfiguriert:

- `isMinifyEnabled = true` — R8/ProGuard an, ungenutzter Code entfernt.
- `isShrinkResources = true` — ungenutzte Ressourcen entfernt.
- `signingConfig` mit Keystore `meteo-release-new.jks` (unter `android_app/app/`).

Release‑Build:

```bash
cd android_app
./gradlew assembleRelease
```

Signiertes Release‑APK:

```text
android_app/app/build/outputs/apk/release/app-release.apk
```

Release‑APK ist deutlich kleiner (~3 MB vs ~18 MB dank R8 + shrinkResources).

### Installation auf Gerät (ADB)

```bash
cd android_app
adb install -r app/build/outputs/apk/release/app-release.apk
```

Ohne `adb` im PATH den vollen Pfad zu `adb.exe` nutzen.

---

## Verhalten der Android‑App

### Betriebsarten der Station

- **BLE‑Modus** (Standard nach Flash):
  - ESP32 wirbt als `MeteoStation`.
  - App findet das Gerät über BLE und empfängt Wetter‑JSON.

- **Wi‑Fi STA‑Modus**:
  - Board verbindet sich mit Router mit den erhaltenen Parametern.
  - Daten über HTTP: Web‑UI (`/`) und Metriken (`/metrics`).
  - Aktuelle IP wird per BLE über `wifi_status` gemeldet.

### UI der Android‑App

Angezeigt werden:

- Aktuelle Messwerte:
  - Temperatur (°C), Luftfeuchte (%), Druck (hPa), Höhe (m).
- Verbindungsstatus: verbunden / verbindet… / getrennt.
- Aktueller Modus und Wi‑Fi‑Status.

Top‑Bar zeigt:

- Titel `MeteoStation`.
- Rechts ein Modus‑Icon:
  - **Bluetooth‑Icon** — Station im BLE oder ohne aktives Wi‑Fi.
  - **Wi‑Fi‑Icon** — Station im Wi‑Fi STA und erfolgreich verbunden.

Icon‑Zustand basiert auf:

- `connectionMode` (BLE / WIFI_STA) aus dem Repository.
- `isWifiConnected`, aktualisiert bei `wifi_status` von der ESP32.

### Interaktion mit der ESP32

- App abonniert BLE‑Charakteristik der Messungen.
- Updates kommen als JSON und werden geparst.
- `mode_changed` und `wifi_status` aktualisieren StateFlow im Repository.
- `MeteoViewModel` liest StateFlow und füttert `MeteoUiState` für das UI.

Überflüssige `Log.d`/`Log.v` entfernt, verbleiben `Log.w`/`Log.e` und nutzerrelevante Meldungen.

---

## BLE- und HTTP‑Interfaces

### BLE‑UUID

- Service: `4b18459e-2b1e-4b5e-ac35-3925df7391b0`
- Daten‑Charakteristik: `b1e6e34c-f61b-4b11-a2c7-0a20f47e5442`
- Modus‑Charakteristik (falls separat): `c1218384-3fa0-4a9d-8f4d-b9d8f9363a7d`

### Wi‑Fi STA

- Nutzt SSID/Passwort, die per BLE vom Telefon kommen.
- Bei Erfolg sendet `wifi_status` mit `status="ok"` und `ip`.
- Bei Fehler `wifi_status` mit `status="error"` und `reason`.

---

## Veröffentlichung im Google Play (kurz)

- Target **SDK 34**, nur notwendige Berechtigungen (BLE, Standort, Netzwerk).
- Release‑Build ist mit eigenem Keystore signiert.
- Für die Veröffentlichung wird eine **Privacy Policy** benötigt (Beispiel: `android_app/privacy_policy.md`).

---

## Lizenz

Firmware und zugehörige Binärdateien sind ausschließlich für den privaten, nichtkommerziellen Gebrauch bestimmt.

Erlaubt:

- Nutzung der Firmware auf eigenen Geräten für private Zwecke;
- Modifikation für den persönlichen Gebrauch;
- Weitergabe unveränderter Binärdateien an andere, ebenfalls nur privat.

Verboten ohne schriftliche Zustimmung des Rechteinhabers:

- Nutzung der Firmware oder Modifikationen in kommerziellen Produkten und Services;
- Verkauf der Firmware (einzeln oder in Geräten);
- Einbindung der Firmware in kostenpflichtige Lösungen.

Alle Rechte bleiben beim Autor.

---

# MeteoStation — Українська

Метеостанція на базі **ESP32‑C3** із датчиками **AHT20** + **BMP280** та Android‑додатком на Jetpack Compose.

Станція підтримує два режими:

- **BLE (Bluetooth Low Energy)** — початкове налаштування та передача метеоданих через BLE.
- **Wi‑Fi STA (клієнт)** — плата підключається до роутера й віддає дані по HTTP, доступна веб‑сторінка та JSON‑метрики.

Android‑додаток підключається до станції, показує вимірювання, дозволяє перемикати режими та налаштовувати Wi‑Fi **без зміни прошивки**.

---

## Структура репозиторію

- **`firmware/`** — прошивка для ESP32‑C3 (Arduino / платформа ESP32):
  - Два режими роботи: **BLE** і **Wi‑Fi STA**.
  - Автовизначення датчиків на I2C (AHT20, BMP280).
  - Передача метеоданих у JSON:
    - по BLE (сповіщення характеристики);
    - по HTTP (`/metrics`) у Wi‑Fi режимі.
  - Вбудований веб‑сервер із простим HTML/JS UI (`/`).
  - Службові JSON‑повідомлення по BLE про зміну режиму та статус Wi‑Fi.

- **`android_app/`** — Android‑додаток (Kotlin + Jetpack Compose):
  - Взаємодія зі станцією по BLE і HTTP.
  - Автооновлення іконки режиму (BLE / Wi‑Fi STA).
  - Відображення температури, вологості, тиску, висоти.
  - Сканування Wi‑Fi мереж, вибір SSID і введення пароля.
  - Надсилання Wi‑Fi налаштувань на плату по BLE, без прошивки.
  - Оптимізований **release APK** (R8 + shrinkResources, мінімум логів).

---

## Прошивка ESP32‑C3

### Джерела

Основний скетч:

```text
firmware/meteo_station/meteo_station.ino
```

Відкрити в Arduino IDE або PlatformIO.

### Необхідні бібліотеки

- `NimBLE-Arduino`
- `Adafruit_AHTX0`
- `Adafruit_BMP280`

### Налаштування Wi‑Fi

Раніше SSID/пароль задавалися константами (`WIFI_STA_SSID`, `WIFI_STA_PASSWORD`). Тепер **Wi‑Fi налаштовується з Android‑додатка по BLE**:

- Плата стартує в BLE‑режимі.
- Додаток підключається по BLE.
- Користувач обирає Wi‑Fi та вводить пароль.
- Додаток надсилає JSON із налаштуваннями на ESP32.
- ESP32 перемикається в Wi‑Fi STA, підключається до роутера і надсилає статус.

Прошивку більше не потрібно змінювати під кожного користувача.

### Збірка та прошивання

1. Встановіть підтримку плат **ESP32** в Arduino IDE.
2. Відкрийте `firmware/meteo_station/meteo_station.ino`.
3. Оберіть плату **LOLIN C3 Mini** / **ESP32‑C3**.
4. Скомпілюйте й прошийте плату.

Після успішного підключення до Wi‑Fi станція отримує IP. За цією IP доступні:

- Веб‑сторінка: `http://<ip_адреса>/`
- JSON‑метрики: `http://<ip_адреса>/metrics`

Додатково ESP32 надсилає по BLE службові JSON‑повідомлення:

- `{"type":"wifi_status","status":"ok","ip":"..."}` — успішне підключення.
- `{"type":"wifi_status","status":"error","reason":"..."}` — помилка.
- `{"type":"mode_changed","mode":"BLE"|"STA"}` — зміна режиму.

Зайвий Serial‑лог вимкнено прапорцем `kDebugSerialOutput = false`, лишаються тільки метео‑JSON і відповіді.

---

## Збірка Android‑додатка

### Вимоги

- Android Studio з Kotlin і Jetpack Compose.
- JDK 17.

### Основні технології

- Kotlin, Coroutines, StateFlow.
- Jetpack Compose, Material3.
- OkHttp для HTTP‑запитів.
- `kotlinx.serialization` для JSON.

Версії вказані в `android_app/app/build.gradle.kts` (Kotlin 1.9.x, Compose BOM `2023.10.01`).

### Debug‑збірка

```bash
cd android_app
./gradlew assembleDebug
```

APK буде за шляхом:

```text
android_app/app/build/outputs/apk/debug/app-debug.apk
```

### Release‑збірка (оптимізований APK)

У `android_app/app/build.gradle.kts` налаштований release‑профіль:

- `isMinifyEnabled = true` — R8/ProGuard, видаляється непотрібний код.
- `isShrinkResources = true` — видаляються невикористані ресурси.
- `signingConfig` з keystore `meteo-release-new.jks` (у `android_app/app/`).

Збірка:

```bash
cd android_app
./gradlew assembleRelease
```

Підписаний release‑APK:

```text
android_app/app/build/outputs/apk/release/app-release.apk
```

Розмір release‑APK помітно менший (~3 МБ проти ~18 МБ завдяки R8 + shrinkResources).

### Встановлення на пристрій (ADB)

```bash
cd android_app
adb install -r app/build/outputs/apk/release/app-release.apk
```

Якщо `adb` не в PATH — використовуйте повний шлях до `adb.exe`.

---

## Поведінка Android‑додатка

### Режими станції

- **BLE режим** (за замовчуванням після прошивки):
  - ESP32 рекламується як `MeteoStation`.
  - Додаток знаходить пристрій по BLE і отримує метеодані в JSON.

- **Wi‑Fi STA режим**:
  - Плата підключається до роутера з параметрами від додатка.
  - Дані віддаються по HTTP: веб‑інтерфейс (`/`) і метрики (`/metrics`).
  - Поточна IP станції передається по BLE через `wifi_status`.

### UI додатка

Показує:

- Поточні метеопоказники: температура, вологість, тиск, висота.
- Статус підключення: підключено / підключення… / відключено.
- Поточний режим і стан Wi‑Fi.

Верхня панель:

- Назва `MeteoStation`.
- Іконка режиму справа:
  - **Bluetooth‑іконка** — станція в BLE або без активного Wi‑Fi.
  - **Wi‑Fi‑іконка** — станція у Wi‑Fi STA і успішно підключена.

Розрахунок іконки базується на `connectionMode` (BLE / WIFI_STA) та `isWifiConnected`, що оновлюється при `wifi_status` від ESP32.

### Взаємодія з ESP32

- Додаток підписується на BLE‑характеристику вимірювань.
- Оновлення приходять як JSON і парсяться в модель.
- `mode_changed` і `wifi_status` оновлюють StateFlow у репозиторії.
- `MeteoViewModel` читає StateFlow і формує `MeteoUiState` для UI.

Зайві `Log.d`/`Log.v` прибрані, залишені `Log.w`/`Log.e` та важливі повідомлення.

---

## BLE та HTTP інтерфейси

### BLE UUID

- Сервіс: `4b18459e-2b1e-4b5e-ac35-3925df7391b0`
- Характеристика даних: `b1e6e34c-f61b-4b11-a2c7-0a20f47e5442`
- Характеристика режиму (якщо окрема): `c1218384-3fa0-4a9d-8f4d-b9d8f9363a7d`

### Wi‑Fi STA

- Використовує SSID/пароль, передані з телефона по BLE.
- При успіху надсилає `wifi_status` з `status="ok"` і `ip`.
- При помилці — `wifi_status` з `status="error"` і `reason`.

---

## Публікація в Google Play (коротко)

- Таргет **SDK 34**, лише необхідні дозволи (BLE, локація, мережа).
- Release‑збірка підписана власним keystore.
- Для публікації потрібна **Privacy Policy** (приклад: `android_app/privacy_policy.md`).

---

## Ліцензія

Прошивки та бінарні файли надаються тільки для особистого, некомерційного використання.

Дозволяється:

- використовувати прошивки на своїх пристроях у особистих цілях;
- модифікувати їх для особистого використання;
- ділитися незміненими бінарниками з іншими користувачами, також лише для особистого використання.

Заборонено без письмового дозволу правовласника:

- використовувати прошивки або модифікації в комерційних продуктах і послугах;
- продавати прошивки (окремо або у складі пристроїв);
- включати прошивки до будь‑яких платних рішень.

Усі права на матеріали зберігаються за автором.

---

# MeteoStation — English

Weather station based on **ESP32‑C3** with **AHT20** + **BMP280** sensors and an Android app built with Jetpack Compose.

The station supports two modes:

- **BLE (Bluetooth Low Energy)** — initial setup and weather data transfer over BLE.
- **Wi‑Fi STA (client)** — the board connects to a router and serves data over HTTP, with a web page and JSON metrics.

The Android app connects to the station, shows readings, lets you switch modes, and configure Wi‑Fi **without changing the firmware**.

---

## Repository structure

- **`firmware/`** — ESP32‑C3 firmware (Arduino / ESP32 platform):
  - Two operating modes: **BLE** and **Wi‑Fi STA**.
  - Auto‑detect sensors on I2C (AHT20, BMP280).
  - Weather data in JSON:
    - via BLE (characteristic notifications);
    - via HTTP (`/metrics`) in Wi‑Fi mode.
  - Built‑in web server with simple HTML/JS UI (`/`).
  - Service JSON messages over BLE about mode changes and Wi‑Fi status.

- **`android_app/`** — Android app (Kotlin + Jetpack Compose):
  - Interacts with the station via BLE and HTTP.
  - Auto‑updates the mode icon (BLE / Wi‑Fi STA).
  - Displays temperature, humidity, pressure, altitude.
  - Wi‑Fi scan, SSID selection, and password entry.
  - Sends Wi‑Fi settings to the board over BLE, no reflashing.
  - Optimized **release APK** (R8 + shrinkResources, minimal logs).

---

## ESP32‑C3 firmware

### Sources

Main sketch:

```text
firmware/meteo_station/meteo_station.ino
```

Open in Arduino IDE or PlatformIO.

### Required libraries

- `NimBLE-Arduino`
- `Adafruit_AHTX0`
- `Adafruit_BMP280`

### Wi‑Fi setup

Previously SSID/password were constants (`WIFI_STA_SSID`, `WIFI_STA_PASSWORD`). Now **Wi‑Fi is configured from the Android app via BLE**:

- Board starts in BLE mode.
- App connects over BLE.
- User selects Wi‑Fi and enters password.
- App sends JSON settings to ESP32.
- ESP32 switches to Wi‑Fi STA, connects to the router, and sends status back.

No need to edit firmware per user anymore.

### Build and flash

1. Install **ESP32** boards in Arduino IDE.
2. Open `firmware/meteo_station/meteo_station.ino`.
3. Select **LOLIN C3 Mini** / **ESP32‑C3** board.
4. Build and upload to the board.

After Wi‑Fi connection, the station gets an IP. Available at:

- Web page: `http://<ip_address>/`
- JSON metrics: `http://<ip_address>/metrics`

ESP32 also sends BLE service JSON messages:

- `{"type":"wifi_status","status":"ok","ip":"..."}` — Wi‑Fi connected.
- `{"type":"wifi_status","status":"error","reason":"..."}` — error.
- `{"type":"mode_changed","mode":"BLE"|"STA"}` — mode change.

Extra Serial debug output is disabled with `kDebugSerialOutput = false`; only weather JSON and responses remain.

---

## Building the Android app

### Requirements

- Android Studio with Kotlin and Jetpack Compose.
- JDK 17.

### Key technologies

- Kotlin, Coroutines, StateFlow.
- Jetpack Compose, Material3.
- OkHttp for HTTP requests.
- `kotlinx.serialization` for JSON.

Versions are set in `android_app/app/build.gradle.kts` (Kotlin 1.9.x, Compose BOM `2023.10.01`).

### Debug build

```bash
cd android_app
./gradlew assembleDebug
```

APK path:

```text
android_app/app/build/outputs/apk/debug/app-debug.apk
```

### Release build (optimized APK)

`android_app/app/build.gradle.kts` configures release:

- `isMinifyEnabled = true` — R8/ProGuard enabled.
- `isShrinkResources = true` — unused resources removed.
- `signingConfig` with keystore `meteo-release-new.jks` (under `android_app/app/`).

Build:

```bash
cd android_app
./gradlew assembleRelease
```

Signed release APK:

```text
android_app/app/build/outputs/apk/release/app-release.apk
```

Release APK is much smaller (~3 MB vs ~18 MB thanks to R8 + shrinkResources).

### Install to device (ADB)

```bash
cd android_app
adb install -r app/build/outputs/apk/release/app-release.apk
```

If `adb` isn’t in PATH, use the full path to `adb.exe`.

---

## Android app behavior

### Station modes

- **BLE mode** (default after flashing):
  - ESP32 advertises as `MeteoStation`.
  - App discovers it over BLE and receives weather JSON.

- **Wi‑Fi STA mode**:
  - Board connects to router with app‑provided params.
  - Data over HTTP: web UI (`/`) and metrics (`/metrics`).
  - Current IP sent via BLE `wifi_status`.

### App UI

Shows:

- Current readings: temperature, humidity, pressure, altitude.
- Connection status: connected / connecting… / disconnected.
- Current mode and Wi‑Fi state.

Top bar:

- Title `MeteoStation`.
- Mode icon on the right:
  - **Bluetooth icon** — station in BLE or no active Wi‑Fi.
  - **Wi‑Fi icon** — station in Wi‑Fi STA and connected.

Icon state is based on `connectionMode` (BLE / WIFI_STA) and `isWifiConnected`, updated on `wifi_status` from ESP32.

### Interaction with ESP32

- App subscribes to BLE measurement characteristic.
- Updates arrive as JSON and are parsed.
- `mode_changed` and `wifi_status` update StateFlow in the repository.
- `MeteoViewModel` reads StateFlow and updates `MeteoUiState` for UI.

Superfluous `Log.d`/`Log.v` removed; kept `Log.w`/`Log.e` and user‑relevant messages.

---

## BLE and HTTP interfaces

### BLE UUID

- Service: `4b18459e-2b1e-4b5e-ac35-3925df7391b0`
- Data characteristic: `b1e6e34c-f61b-4b11-a2c7-0a20f47e5442`
- Mode characteristic (if separate): `c1218384-3fa0-4a9d-8f4d-b9d8f9363a7d`

### Wi‑Fi STA

- Uses SSID/password sent from phone via BLE.
- On success sends `wifi_status` with `status="ok"` and `ip`.
- On error sends `wifi_status` with `status="error"` and `reason`.

---

## Publishing to Google Play (brief)

- Targets **SDK 34**, only necessary permissions (BLE, location, network).
- Release build is signed with own keystore.
- Requires a **Privacy Policy** page (example: `android_app/privacy_policy.md`).

---

## License

Firmware and binaries are for personal, non‑commercial use only.

Allowed:

- use firmware on your own devices for personal purposes;
- modify it for personal use;
- share unmodified binaries with others for personal use only.

Forbidden without prior written permission of the rights holder:

- use firmware or modifications in commercial products or services;
- sell firmware (alone or in devices);
- include firmware in paid solutions or offers.

All rights to firmware and materials remain with the author.

---

# MeteoStation — Русский (оригинал)

Метеостанция на базе **ESP32‑C3** с датчиком **AHT20** + **BMP280** и Android‑приложением на Jetpack Compose.

Станция умеет работать в двух режимах:

- **BLE (Bluetooth Low Energy)** — начальная настройка и передача метеоданных по BLE.
- **Wi‑Fi STA (клиент)** — плата подключается к роутеру и отдаёт данные по HTTP, доступна веб‑страница и JSON‑метрики.

Android‑приложение подключается к станции, показывает текущие измерения, позволяет переключать режимы и настраивать Wi‑Fi **без правки прошивки**.

---

## Структура репозитория

- **`firmware/`** — прошивка для ESP32‑C3 (Arduino / платформа ESP32):
  - Поддержка двух режимов работы: **BLE** и **Wi‑Fi STA**.
  - Автоопределение датчиков на шине I2C (AHT20, BMP280).
  - Передача метеоданных в JSON:
    - по BLE (нотификации характеристики);
    - по HTTP (`/metrics`) в Wi‑Fi режиме.
  - Встроенный веб‑сервер с простым HTML/JS UI (`/`).
  - Служебные JSON‑сообщения по BLE о смене режима и статусе Wi‑Fi.

- **`android_app/`** — Android‑приложение (Kotlin + Jetpack Compose):
  - Взаимодействие со станцией по BLE и по HTTP.
  - Автоматическое обновление иконки режима (BLE / Wi‑Fi STA).
  - Отображение температуры, влажности, давления и высоты.
  - Сканирование Wi‑Fi сетей, выбор SSID и ввод пароля.
  - Отправка Wi‑Fi настроек на плату по BLE, без прошивки.
  - Оптимизированный **release APK** (R8 + shrinkResources, минимум логов).

---

## Прошивка ESP32‑C3

### Исходники

Основной скетч:

```text
firmware/meteo_station/meteo_station.ino
```

Открыть в Arduino IDE или PlatformIO.

### Необходимые библиотеки

- `NimBLE-Arduino`
- `Adafruit_AHTX0`
- `Adafruit_BMP280`

### Настройка Wi‑Fi

Ранее SSID/пароль задавались константами в прошивке (`WIFI_STA_SSID`, `WIFI_STA_PASSWORD`).
Сейчас **Wi‑Fi настраивается из Android‑приложения по BLE**:

- Плата стартует в BLE‑режиме.
- Приложение подключается по BLE.
- Пользователь выбирает Wi‑Fi сеть и вводит пароль.
- Приложение отправляет JSON с настройками на ESP32.
- ESP32 переключается в Wi‑Fi STA, подключается к роутеру и отсылает статус.

Прошивка больше не требует изменения SSID/пароля в исходниках под каждого пользователя.

### Сборка и заливка

1. Установите поддержку плат **ESP32** в Arduino IDE.
2. Откройте `firmware/meteo_station/meteo_station.ino`.
3. Выберите плату **LOLIN C3 Mini** / **ESP32‑C3** (соответствующий профиль).
4. Скомпилируйте и загрузите прошивку на плату.

После успешного подключения к Wi‑Fi станция получает IP‑адрес от роутера. По этому IP доступны:

- Веб‑страница: `http://<ip_адрес>/`
- JSON‑метрики: `http://<ip_адрес>/metrics`

Дополнительно ESP32 отправляет по BLE служебные JSON‑сообщения:

- `{"type":"wifi_status","status":"ok","ip":"..."}` — успешное подключение к Wi‑Fi.
- `{"type":"wifi_status","status":"error","reason":"..."}` — ошибка подключения.
- `{"type":"mode_changed","mode":"BLE"|"STA"}` — смена режима работы.

Лишний отладочный вывод в Serial отключён флагом `kDebugSerialOutput = false`, чтобы в терминал уходили только полезные данные (метео‑JSON и ответы на команды).

---

## Сборка Android‑приложения

### Требования

- Android Studio с поддержкой Kotlin и Jetpack Compose.
- JDK 17.

### Основные технологии

- Kotlin, Coroutines, StateFlow.
- Jetpack Compose, Material3.
- OkHttp для HTTP‑запросов.
- `kotlinx.serialization` для JSON.

Зависимости и версии заданы в `android_app/app/build.gradle.kts` и актуальны для Kotlin 1.9.x и Compose BOM `2023.10.01`.

### Debug сборка

```bash
cd android_app
./gradlew assembleDebug
```

APK будет находиться по пути:

```text
android_app/app/build/outputs/apk/debug/app-debug.apk
```

### Release сборка (оптимизированный APK)

В `android_app/app/build.gradle.kts` настроен release‑профиль:

- `isMinifyEnabled = true` — включён R8/ProGuard, вырезается неиспользуемый код.
- `isShrinkResources = true` — вырезаются неиспользуемые ресурсы.
- Настроен `signingConfig` c keystore `meteo-release-new.jks` (находится в `android_app/app/`).

Сборка release APK:

```bash
cd android_app
./gradlew assembleRelease
```

Подписанный релизный APK:

```text
android_app/app/build/outputs/apk/release/app-release.apk
```

Размер релизного APK значительно меньше debug (порядка ~3 МБ против ~18 МБ за счёт R8 и shrinkResources).

### Установка на устройство (ADB)

```bash
cd android_app
adb install -r app/build/outputs/apk/release/app-release.apk
```

При отсутствии `adb` в PATH можно использовать полный путь к `adb.exe`.

---

## Поведение Android‑приложения

### Режимы работы станции

- **BLE режим** (по умолчанию после прошивки):
  - ESP32 рекламируется как `MeteoStation`.
  - Android‑приложение находит устройство по BLE и получает метеоданные в JSON.

- **Wi‑Fi STA режим**:
  - Плата подключается к роутеру с параметрами, полученными от приложения.
  - Данные отдаются по HTTP: веб‑интерфейс (`/`) и метрики (`/metrics`).
  - Текущий IP станции передаётся в Android‑приложение по BLE через сообщения `wifi_status`.

### UI Android‑приложения

Приложение отображает:

- Текущие метеопоказания:
  - Температура (°C).
  - Влажность (%).
  - Давление (гПа).
  - Высота (м).
- Статус подключения: подключено / подключение… / отключено.
- Текущий режим станции и состояние Wi‑Fi.

Верхняя панель показывает:

- Название `MeteoStation`.
- Справа — иконку режима подключения:
  - **Bluetooth‑иконка** — станция в BLE‑режиме или нет активного Wi‑Fi.
  - **Wi‑Fi‑иконка** — станция в режиме Wi‑Fi STA и успешно подключена к сети.

Состояние иконки рассчитывается на основе:

- `connectionMode` (BLE / WIFI_STA), приходящего из репозитория.
- `isWifiConnected`, который обновляется при получении `wifi_status` от ESP32.

### Взаимодействие с ESP32

- Приложение подписывается на BLE‑характеристику измерений.
- Обновления приходят в виде JSON и парсятся в модель.
- Изменение режима (`mode_changed`) и статус Wi‑Fi (`wifi_status`) обновляют StateFlow в репозитории.
- `MeteoViewModel` подписывается на эти StateFlow и обновляет `MeteoUiState`, который читает UI.

Лишние `Log.d`/`Log.v` в приложении удалены, оставлены только важные предупреждения/ошибки (`Log.w`/`Log.e`) и сообщения, критичные для пользователя.

---

## BLE и HTTP интерфейсы

### BLE UUID

- Сервис: `4b18459e-2b1e-4b5e-ac35-3925df7391b0`
- Характеристика данных: `b1e6e34c-f61b-4b11-a2c7-0a20f47e5442`
- Характеристика режима (если используется отдельная): `c1218384-3fa0-4a9d-8f4d-b9d8f9363a7d`

### Wi‑Fi STA

- Использует SSID и пароль, переданные с телефона по BLE.
- После успешного подключения отправляет `wifi_status` с `status="ok"` и `ip`.
- При ошибке отправляет `wifi_status` с `status="error"` и `reason`.

---

## Публикация в Google Play (кратко)

- Приложение таргетирует **SDK 34**, использует только необходимые разрешения (BLE, локация, сеть).
- Release‑сборка подписана собственным keystore.
- Для публикации требуется страница **Privacy Policy** (пример: `android_app/privacy_policy.md`).

---

## Лицензия

Прошивки и сопутствующие бинарные файлы в этом репозитории предоставляются
исключительно для личного, некоммерческого использования.

Разрешается:

- использовать прошивки на своих устройствах в личных целях;
- модифицировать прошивки для личного использования;
- делиться неизменёнными бинарными файлами с другими пользователями, также только для личного использования.

Запрещается без предварительного письменного разрешения правообладателя:

- использовать прошивки или их модификации в коммерческих продуктах и услугах;
- продавать прошивки (как отдельно, так и в составе устройств);
- включать прошивки в состав любых платных решений или предложений.

Все права на прошивки и связанные материалы сохраняются за автором проекта.
