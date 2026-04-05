# MeteoStation (ESP32‑C3 Firmware)

## Deutsch

Wetterstation auf Basis der **ESP32‑C3** mit den Sensoren **AHT20** + **BMP280** und einer eigenen Android‑App.

Dieses Repository dient **zur Verteilung der fertigen Firmware‑Binärdateien** und zur Beschreibung der Gerätemöglichkeiten. Der Quellcode der Android‑App und Build‑Details werden hier nicht veröffentlicht.

---

## Funktionen der Station

- Zwei Betriebsarten:
  - **BLE (Bluetooth Low Energy)** – Ersteinrichtung und Empfang der Wetterdaten über BLE.
  - **Wi‑Fi STA (Client)** – das Board verbindet sich mit Ihrem Router und liefert Daten per HTTP.
- Automatische Erkennung der Sensoren auf dem I2C‑Bus (**AHT20**, **BMP280**).
- Übertragung der Wetterdaten im **JSON**‑Format:
  - über BLE (Characteristic Notifications);
  - über HTTP (`/metrics`) im Wi‑Fi‑Modus.
- Eingebauter Webserver:
  - Weboberfläche unter `http://<ip_adresse>/`;
  - JSON‑Metriken unter `http://<ip_adresse>/metrics`.
- Service‑JSON‑Nachrichten über BLE:
  - Wi‑Fi‑Status (`wifi_status`);
  - Wechsel des Betriebsmodus (`mode_changed`).

Die Android‑App verbindet sich mit der Station, zeigt aktuelle Messwerte an und ermöglicht die Wi‑Fi‑Konfiguration **ohne Firmwareänderung**.

---

## Betriebsarten

### BLE (Bluetooth Low Energy)

- Nach dem Flashen startet die Station im **BLE‑Modus**.
- Die ESP32‑C3 wirbt z. B. unter dem Namen `MeteoStation`.
- Das Telefon (über die App) findet das Gerät, verbindet sich per BLE und:
  - erhält aktuelle Wetterdaten als JSON;
  - kann Wi‑Fi‑Einstellungen an die Station senden.

### Wi‑Fi STA (Client)

- Nach Empfang der Wi‑Fi‑Einstellungen vom Telefon schaltet die Station in den **Wi‑Fi‑STA‑Modus**.
- Sie verbindet sich mit Ihrem Router und erhält eine IP‑Adresse.
- Unter dieser IP sind erreichbar:
  - Web‑Interface: `http://<ip_adresse>/`;
  - JSON‑Metriken: `http://<ip_adresse>/metrics`.
- Verbindungsstatus und IP‑Adresse werden per BLE als Nachricht zurückgesendet:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
```

Bei Verbindungsfehlern wird eine Nachricht mit `status="error"` und `reason` gesendet.

---

## Anschluss der Sensoren AHT20 + BMP280

Beide Sensoren arbeiten über **I2C** und können parallel angeschlossen werden (gemeinsame SDA/SCL‑Leitungen).

### Pins der ESP32‑C3 (LOLIN C3 Mini)

Laut Firmware `meteo_station.ino`:

- `SDA_PIN = 6`
- `SCL_PIN = 7`

Anschluss:

- **AHT20**:
  - `VCC` → 3,3 V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA)
  - `SCL` → GPIO **7** (SCL)

- **BMP280**:
  - `VCC` → 3,3 V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA) — gleiche Leitung wie AHT20
  - `SCL` → GPIO **7** (SCL) — gleiche Leitung wie AHT20

Versorgung nur **3,3 V** (wie bei der ESP32‑C3). Beide Sensoren hängen am selben I2C‑Bus, Adressen laut Firmware:

- `AHT20` — `0x38`
- `BMP280` — `0x76` oder `0x77` (automatisch erkannt)

Es können zwei einzelne Module oder ein **Kombimodul „2‑in‑1“ (AHT20 + BMP280)** verwendet werden. Der Anschluss bleibt identisch: **4 Leitungen** an die ESP32‑C3: 3,3 V, GND, SDA (GPIO 6), SCL (GPIO 7).

---

## Release‑Struktur

Im Bereich **Releases** (oder im Binary‑Ordner) liegen die fertigen Firmware‑Dateien. Für das Board ESP32‑C3 (Profil `esp32.esp32.lolin_c3_mini`) werden genutzt:

- `meteo_station.ino.bootloader.bin` — Bootloader
- `meteo_station.ino.partitions.bin` — Partitionstabelle
- `meteo_station.ino.bin` — Hauptfirmware (Applikation)
- `meteo_station.ino.merged.bin` — **kombinierter Binärfile**, enthält alles oben genannte

Je nach Bedarf können einzelne Dateien oder nur der kombinierte `meteo_station.ino.merged.bin` veröffentlicht werden.

---

## Flashen eines fertigen Binärs

> ACHTUNG: Flashen erfolgt auf eigene Gefahr. Vergewissern Sie sich, dass Sie den richtigen Port und das richtige Board auswählen.

Für die ESP32‑C3 können Sie nutzen:

- Offizielles Tool **ESP32 Flash Download Tool** (GUI von Espressif).
- Oder die CLI‑Utility **esptool.py**.

### Variante 1: ESP32 Flash Download Tool (GUI)

1. Laden Sie „ESP32 Flash Download Tool“ von der Espressif‑Website:

   <https://docs.espressif.com/projects/esp-test-tools/en/latest/esp32/production_stage/tools/flash_download_tool.html>

   Oder verwenden Sie das im Repository beiliegende Archiv `flash_download_tool.zip`.
2. Schließen Sie das ESP32‑C3‑Board per USB an den PC an.
3. Versetzen Sie das Board in den Boot‑Modus (Boot/EN – siehe Board‑Doku).
4. Wählen Sie im Tool den Chip **ESP32‑C3**.
5. Geben Sie Binärdateien und Adressen an. Typische Konfiguration:
   - `meteo_station.ino.bootloader.bin` → `0x0000`
   - `meteo_station.ino.partitions.bin` → `0x8000`
   - `meteo_station.ino.bin` → `0x10000`
6. Wählen Sie den COM‑Port Ihres Boards.
7. Klicken Sie **START** und warten Sie bis zum Ende des Flash‑Vorgangs.

Exakte Adressen und Dateisätze immer mit der Beschreibung des jeweiligen Releases abgleichen — sie können abweichen.

#### 1.2. Flashen der kombinierten Binärdatei

Wenn der Release die Datei `meteo_station.ino.merged.bin` enthält, sind Bootloader, Partitionen und Hauptfirmware bereits zusammengefügt. In diesem Fall genügt es, **eine Datei** ab Adresse `0x0` zu flashen:

- Stellen Sie sicher, dass Sie die `meteo_station.ino.merged.bin` aus dem passenden Release verwenden.

---

## Kommunikationsprotokoll (kurz)

### BLE‑JSON‑Nachrichten

- **Wetterdaten** – periodische Benachrichtigungen mit Temperatur, Luftfeuchtigkeit, Druck, Höhe.
- **Wi‑Fi‑Status**:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
{"type":"wifi_status","status":"error","reason":"wrong_password"}
```

- **Moduswechsel**:

```json
{"type":"mode_changed","mode":"BLE"}
{"type":"mode_changed","mode":"STA"}
```

### HTTP‑Interface

- Web‑Seite: `http://<ip_adresse>/`
- Metriken: `http://<ip_adresse>/metrics` (JSON)

---

## Web‑Oberfläche

Die Station bietet eine umfangreiche Web‑Oberfläche mit folgenden Funktionen:

### Hauptseite

- Anzeige der aktuellen Messwerte (Temperatur, Luftfeuchtigkeit, Druck, Höhe)
- Statusanzeige (Online, Wi‑Fi verbunden)

### Menü‑Schaltflächen

- **📊 Diagramme** — Echtzeit‑Diagramme der Sensordaten (aktualisieren sich jede Sekunde)
- **📈 Statistik** — Verlaufsanzeige mit konfigurierbarem Zeitraum (1 Stunde bis alle Daten)
- **🌐 Sprache** — Auswahl der Benutzeroberflächen‑Sprache (4 Sprachen verfügbar)
- **⚙️ Einheiten** — Einstellung der Maßeinheiten und Korrekturwerte
- **📡 BT Modus** — Zurücksetzen in den BLE‑Modus (Neustart erforderlich)

### Spracheinstellung

Die Web‑Oberfläche unterstützt 4 Sprachen:
- 🇷🇺 Русский (Russisch)
- 🇬🇧 English (Englisch)
- 🇺🇦 Українська (Ukrainisch)
- 🇩🇪 Deutsch

Die Spracheinstellung wird gespeichert und nach einem Neustart beibehalten.

### Maßeinheiten

Im Menü **Einheiten** können Sie wählen:
- **Temperatur**: °C oder °F
- **Druck**: hPa, kPa oder mmHg
- **Höhe**: Meter oder Fuß

### Korrekturwerte

Wenn die Sensoren systematische Abweichungen zeigen, können Sie im selben Menü Korrekturwerte eingeben:
- Temperatur (Schritt 0,1 °C)
- Luftfeuchtigkeit (Schritt 0,1 %)
- Luftdruck (Schritt 0,1 hPa)
- Höhe (Schritt 0,1 m)

Die Korrekturwerte werden auf alle angezeigten Daten angewendet und gespeichert.

### BT‑Modus‑Schaltfläche

Die Schaltfläche **📡 BT Modus** setzt die Station in den BLE‑Betriebsmodus zurück:

- Nach Bestätigung startet die Station neu
- Nach dem Neustart arbeitet die Station im BLE‑Modus
- Wi‑Fi‑Einstellungen bleiben erhalten (werden nicht gelöscht)

Dies ist nützlich, wenn Sie die Station erneut über die Android‑App konfigurieren möchten.

### Hardware‑Taste: Werksreset

Auf dem Board befindet sich die **BOOT‑Taste** (GPIO 9). Sie ermöglicht einen vollständigen Werksreset:

- Halten Sie die BOOT‑Taste **3 Sekunden** lang gedrückt
- Die Station löscht alle Einstellungen (Wi‑Fi, Sprache, Einheiten, Korrekturwerte)
- Nach dem Löschen startet die Station neu
- Nach dem Neustart arbeitet die Station im BLE‑Modus (wie nach dem ersten Flashen)

Der Werksreset ist nützlich, wenn Sie alle Einstellungen vollständig zurücksetzen möchten.

### Firmware‑Update über Android‑App

Die neue Version der Android‑App ermöglicht das **Flashen und Aktualisieren der Firmware** direkt über ein **USB‑OTG‑Kabel**:

- Verbinden Sie die Station über USB‑OTG mit dem Android‑Gerät
- Die App erkennt die Station automatisch
- Wählen Sie die gewünschte Firmware‑Version
- Der Flash‑Vorgang erfolgt direkt über die App

Dies ersetzt nicht die BLE‑Konfiguration, sondern bietet eine bequeme Methode zur Firmware‑Aktualisierung ohne PC.

---

## Logs und Performance

- Überflüssige serielle Debug‑Ausgaben sind deaktiviert.
- Im Log bleiben nur nutzerrelevante Daten:
  - Wetter‑JSON;
  - Antworten auf Befehle;
  - Schlüsselnachrichten zum Status.

Das verringert die Portlast und vereinfacht die Analyse.

---

## Android‑App

Für die Station wird eine separate Android‑App verwendet:

- Verbindung zur Station per BLE.
- Wi‑Fi‑Konfiguration für den STA‑Modus.
- Umschalten der Modi BLE / Wi‑Fi.
- Anzeige der aktuellen Wetterdaten.

Die App wird separat (über Google Play) verteilt und in diesem Repository **weder gebaut noch im Quellcode veröffentlicht**.

---

## Lizenz

Firmware und zugehörige Binärdateien in diesem Repository werden ausschließlich für den persönlichen, nichtkommerziellen Gebrauch bereitgestellt.

Erlaubt ist:

- Nutzung der Firmware auf eigenen Geräten für private Zwecke;
- Modifikation der Firmware für den persönlichen Gebrauch;
- Weitergabe unveränderter Binärdateien an andere Nutzer, ebenfalls nur für private Zwecke.

Untersagt ohne vorherige schriftliche Zustimmung des Rechteinhabers:

- Nutzung der Firmware oder ihrer Modifikationen in kommerziellen Produkten und Dienstleistungen;
- Verkauf der Firmware (einzeln oder als Teil von Geräten);
- Einbindung der Firmware in kostenpflichtige Lösungen oder Angebote.

Alle Rechte an Firmware und zugehörigen Materialien verbleiben beim Autor des Projekts.

---

## Українська

Метеостанція на базі **ESP32‑C3** із датчиками **AHT20** + **BMP280** та фірмовим Android‑додатком.

Цей репозиторій призначено **для поширення готових прошивок (бінарних файлів)** і опису можливостей пристрою. Вихідний код Android‑додатка та деталі його збирання тут не публікуються.

---

## Можливості станції

- Підтримка двох режимів роботи:
  - **BLE (Bluetooth Low Energy)** — початкове налаштування та отримання метеоданих по BLE.
  - **Wi‑Fi STA (клієнт)** — плата підключається до вашого роутера і віддає дані через HTTP.
- Автовизначення датчиків на шині I2C (**AHT20**, **BMP280**).
- Передача метеоданих у форматі **JSON**:
  - по BLE (сповіщення характеристики);
  - по HTTP (`/metrics`) у Wi‑Fi режимі.
- Вбудований веб‑сервер:
  - веб‑сторінка за адресою `http://<ip_адреса>/`;
  - JSON‑метрики за адресою `http://<ip_адреса>/metrics`.
- Службові JSON‑повідомлення по BLE:
  - статус Wi‑Fi (`wifi_status`);
  - зміна режиму роботи (`mode_changed`).

Android‑додаток підключається до станції, показує поточні вимірювання та дозволяє налаштовувати Wi‑Fi **без зміни прошивки**.

---

## Режими роботи

### BLE (Bluetooth Low Energy)

- Після прошивки станція стартує в **BLE‑режимі**.
- ESP32‑C3 рекламується з іменем, наприклад, `MeteoStation`.
- Телефон (через додаток) знаходить пристрій, підключається по BLE і:
  - отримує поточні метеодані у JSON;
  - може надіслати налаштування Wi‑Fi на станцію.

### Wi‑Fi STA (клієнт)

- Після отримання налаштувань Wi‑Fi від телефону станція перемикається в режим **Wi‑Fi STA**.
- Підключається до вашого роутера й отримує IP‑адресу.
- За цією IP доступні:
  - веб‑інтерфейс: `http://<ip_адреса>/`;
  - метрики у форматі JSON: `http://<ip_адреса>/metrics`.
- Статус підключення та IP‑адреса надсилаються назад у додаток по BLE повідомленням:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
```

У разі помилки підключення надсилається повідомлення з `status="error"` та `reason`.

---

## Підключення датчиків AHT20 + BMP280

Обидва датчики працюють по **I2C** і можуть бути підключені паралельно (спільна шина SDA/SCL).

### Виводи ESP32‑C3 (LOLIN C3 Mini)

За прошивкою `meteo_station.ino`:

- `SDA_PIN = 6`
- `SCL_PIN = 7`

Підключення:

- **AHT20**:
  - `VCC` → 3.3V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA)
  - `SCL` → GPIO **7** (SCL)

- **BMP280**:
  - `VCC` → 3.3V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA) — на ту саму лінію, що й AHT20
  - `SCL` → GPIO **7** (SCL) — на ту саму лінію, що й AHT20

Живлення датчиків — **строго 3.3V** (як у ESP32‑C3). Обидва датчики висять на одній I2C‑шині, їхні адреси вказані у прошивці:

- `AHT20` — `0x38`
- `BMP280` — `0x76` або `0x77` (автовизначення прошивкою)

Можна використовувати окремі модулі AHT20 і BMP280 або **комбінований модуль "2‑в‑1" (AHT20 + BMP280)**. Підключення однакове — лише **4 дроти** до ESP32‑C3: 3.3V, GND, SDA (GPIO 6) і SCL (GPIO 7).

---

## Структура релізів

У розділі **Releases** (або в теці з бінарниками) викладаються готові прошивки. Для плати ESP32‑C3 (профіль `esp32.esp32.lolin_c3_mini`) використовуються файли:

- `meteo_station.ino.bootloader.bin` — завантажувач (bootloader)
- `meteo_station.ino.partitions.bin` — таблиця розділів (partitions)
- `meteo_station.ino.bin` — основна прошивка (додаток)
- `meteo_station.ino.merged.bin` — **комбінований бінарник**, що містить усе перелічене

У релізах можна публікувати як окремі файли, так і лише комбінований `meteo_station.ino.merged.bin` — залежно від зручності користувачів.

---

## Прошивання готового бінарника

> УВАГА: усі дії з прошивкою ви виконуєте на свій страх і ризик. Переконайтесь, що обрали правильний порт і модель плати.

Для прошивки ESP32‑C3 можна використовувати:

- Офіційну утиліту **ESP32 Flash Download Tool** (GUI від Espressif).
- Або консольну утиліту **esptool.py**.

### Варіант 1: ESP32 Flash Download Tool (GUI)

1. Завантажте "ESP32 Flash Download Tool" з сайту Espressif:

   <https://docs.espressif.com/projects/esp-test-tools/en/latest/esp32/production_stage/tools/flash_download_tool.html>

   Або використайте архів `flash_download_tool.zip`, доданий у цьому репозиторії.
2. Підключіть плату ESP32‑C3 до ПК через USB.
3. Переведіть плату в режим завантаження (Boot/EN — див. документацію на вашу плату).
4. В утиліті виберіть потрібний чип (**ESP32‑C3**).
5. Вкажіть бінарні файли та адреси, використовуючи файли з релізу. Для типової конфігурації:
   - `meteo_station.ino.bootloader.bin` → `0x0000`
   - `meteo_station.ino.partitions.bin` → `0x8000`
   - `meteo_station.ino.bin` → `0x10000`
6. Оберіть COM‑порт плати.
7. Натисніть **START** і дочекайтеся завершення прошивки.

Точні адреси й склад бінарників обов’язково звіряйте з описом конкретного релізу — вони можуть відрізнятися.

#### 1.2. Прошивання комбінованого бінарника

Якщо в релізі є файл `meteo_station.ino.merged.bin`, він уже містить bootloader, partitions і основну прошивку, зібрані разом. У такому разі достатньо прошити **один файл** з адреси `0x0`:

- Переконайтесь, що використовуєте саме `meteo_station.ino.merged.bin` із відповідного релізу.

---

## Протокол обміну (коротко)

### BLE JSON повідомлення

- **Метеодані** — періодичні сповіщення з температурою, вологістю, тиском, висотою.
- **Статус Wi‑Fi**:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
{"type":"wifi_status","status":"error","reason":"wrong_password"}
```

- **Зміна режиму**:

```json
{"type":"mode_changed","mode":"BLE"}
{"type":"mode_changed","mode":"STA"}
```

### HTTP інтерфейс

- Веб‑сторінка: `http://<ip_адреса>/`
- Метрики: `http://<ip_адреса>/metrics` (JSON)

---

## Веб‑інтерфейс

Станція має розширений веб‑інтерфейс із такими функціями:

### Головна сторінка

- Відображення поточних вимірювань (температура, вологість, тиск, висота)
- Індикатор статусу (Онлайн, Wi‑Fi підключено)

### Меню кнопок

- **📊 Графіки** — графіки в реальному часі (оновлення кожну секунду)
- **📈 Статистика** — перегляд історії з налаштовуваним періодом (від 1 години до всіх даних)
- **🌐 Мова** — вибір мови інтерфейсу (4 мови)
- **⚙️ Одиниці** — налаштування одиниць виміру та коригування
- **📡 BT режим** — повернення в BLE‑режим (потрібно перезавантаження)

### Мова інтерфейсу

Веб‑інтерфейс підтримує 4 мови:

- 🇷🇺 Русский
- 🇬🇧 English
- 🇺🇦 Українська
- 🇩🇪 Deutsch

Налаштування мови зберігається й відновлюється після перезавантаження.

### Одиниці виміру

У меню **Одиниці** можна обрати:

- **Температура**: °C або °F
- **Тиск**: hPa, kPa або mmHg
- **Висота**: метри або фути

### Коригування показань

Якщо датчики мають систематичні похибки, можна ввести поправки:

- Температура (крок 0,1 °C)
- Вологість (крок 0,1 %)
- Тиск (крок 0,1 hPa)
- Висота (крок 0,1 м)

Поправки застосовуються до всіх даних і зберігаються.

### Кнопка BT режим

Кнопка **📡 BT режим** переводить станцію в BLE‑режим:

- Після підтвердження станція перезавантажиться
- Після перезавантаження станція працює в BLE‑режимі
- Налаштування Wi‑Fi зберігаються (не видаляються)

Це корисно, якщо потрібно переналаштувати станцію через Android‑додаток.

### Апаратна кнопка: Скидання до заводських налаштувань

На платі розміщена кнопка **BOOT** (GPIO 9). Вона дозволяє виконати повне скидання:

- Утримуйте кнопку BOOT **3 секунди**
- Станція видалить усі налаштування (Wi‑Fi, мову, одиниці, коригування)
- Після видалення станція перезавантажиться
- Після перезавантаження станція працює в BLE‑режимі (як після першого прошивання)

Скидання корисне, якщо потрібно повністю очистити всі налаштування.

### Оновлення прошивки через Android‑додаток

Нова версія Android‑додатку дозволяє **прошивати й оновлювати прошивку** безпосередньо через **USB‑OTG‑кабель**:

- Підключіть станцію через USB‑OTG до Android‑пристрою
- Додаток автоматично визначить станцію
- Оберіть потрібну версію прошивки
- Процес прошивання відбувається безпосередньо через додаток

Це не замінює BLE‑налаштування, а надає зручний спосіб оновлення прошивки без ПК.

---

## Логи та продуктивність

- У прошивці відключено зайвий відладочний вивід у Serial.
- У логах лишаються тільки дані, корисні користувачу:
  - метео‑JSON;
  - відповіді на команди;
  - ключові повідомлення про стан.

Це зменшує навантаження на порт і спрощує аналіз роботи станції.

---

## Android‑додаток

Для роботи зі станцією використовується окремий Android‑додаток:

- Підключення до станції по BLE.
- Налаштування Wi‑Fi мережі для режиму STA.
- Перемикання режимів BLE / Wi‑Fi.
- Відображення поточних метеоданих.

Додаток поширюється окремо (через Google Play) і в цьому репозиторії **не збирається та не публікується у вихідних кодах**.

---

## Ліцензія

Прошивки та супровідні бінарні файли в цьому репозиторії надаються виключно для особистого, некомерційного використання.

Дозволяється:

- використовувати прошивки на своїх пристроях у особистих цілях;
- модифікувати прошивки для особистого використання;
- ділитися незміненими бінарними файлами з іншими користувачами, також лише для особистого використання.

Забороняється без попереднього письмового дозволу правовласника:

- використовувати прошивки або їх модифікації в комерційних продуктах і послугах;
- продавати прошивки (окремо або в складі пристроїв);
- включати прошивки до будь‑яких платних рішень чи пропозицій.

Усі права на прошивки та пов’язані матеріали зберігаються за автором проєкту.

---

## English

Weather station based on **ESP32‑C3** with **AHT20** + **BMP280** sensors and a branded Android app.

This repository is intended **to distribute ready‑to‑flash firmware binaries** and describe the device capabilities. The Android app source code and build details are not published here.

---

## Station capabilities

- Two operating modes:
  - **BLE (Bluetooth Low Energy)** — initial setup and receiving weather data over BLE.
  - **Wi‑Fi STA (client)** — the board connects to your router and serves data over HTTP.
- Auto‑detect sensors on the I2C bus (**AHT20**, **BMP280**).
- Weather data in **JSON** format:
  - via BLE (characteristic notifications);
  - via HTTP (`/metrics`) in Wi‑Fi mode.
- Built‑in web server:
  - web page at `http://<ip_address>/`;
  - JSON metrics at `http://<ip_address>/metrics`.
- Service JSON messages over BLE:
  - Wi‑Fi status (`wifi_status`);
  - mode change (`mode_changed`).

The Android app connects to the station, shows current readings, and lets you configure Wi‑Fi **without changing the firmware**.

---

## Operating modes

### BLE (Bluetooth Low Energy)

- After flashing, the station starts in **BLE mode**.
- ESP32‑C3 advertises with a name like `MeteoStation`.
- The phone (via the app) discovers the device, connects over BLE and:
  - receives current weather data in JSON;
  - can send Wi‑Fi settings to the station.

### Wi‑Fi STA (client)

- After getting Wi‑Fi settings from the phone, the station switches to **Wi‑Fi STA**.
- It connects to your router and obtains an IP address.
- Available at this IP:
  - web interface: `http://<ip_address>/`;
  - JSON metrics: `http://<ip_address>/metrics`.
- Connection status and IP are sent back to the app over BLE:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
```

On connection error, a message with `status="error"` and `reason` is sent.

---

## Wiring the AHT20 + BMP280 sensors

Both sensors use **I2C** and can be connected in parallel (shared SDA/SCL bus).

### ESP32‑C3 pins (LOLIN C3 Mini)

According to `meteo_station.ino`:

- `SDA_PIN = 6`
- `SCL_PIN = 7`

Connect as follows:

- **AHT20**:
  - `VCC` → 3.3V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA)
  - `SCL` → GPIO **7** (SCL)

- **BMP280**:
  - `VCC` → 3.3V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA) — same line as AHT20
  - `SCL` → GPIO **7** (SCL) — same line as AHT20

Power for the sensors is **strictly 3.3V** (same as ESP32‑C3). Both hang on one I2C bus, addresses set in firmware:

- `AHT20` — `0x38`
- `BMP280` — `0x76` or `0x77` (auto‑detected)

You can use two separate modules or a **combined “2‑in‑1” module (AHT20 + BMP280)**. The wiring is identical — only **4 wires** to ESP32‑C3: 3.3V, GND, SDA (GPIO 6), SCL (GPIO 7).

---

## Release structure

In **Releases** (or the binaries folder) you’ll find ready firmware files. For ESP32‑C3 (profile `esp32.esp32.lolin_c3_mini`) the files are:

- `meteo_station.ino.bootloader.bin` — bootloader
- `meteo_station.ino.partitions.bin` — partition table
- `meteo_station.ino.bin` — main firmware (application)
- `meteo_station.ino.merged.bin` — **combined binary** including all above

You may publish separate files or only the combined `meteo_station.ino.merged.bin` depending on user convenience.

---

## Flashing a ready binary

> WARNING: flashing is at your own risk. Ensure you pick the correct port and board model.

For ESP32‑C3 you can use:

- Official **ESP32 Flash Download Tool** (Espressif GUI).
- Or CLI **esptool.py**.

### Option 1: ESP32 Flash Download Tool (GUI)

1. Download the “ESP32 Flash Download Tool” from Espressif:

   <https://docs.espressif.com/projects/esp-test-tools/en/latest/esp32/production_stage/tools/flash_download_tool.html>

   Or use the `flash_download_tool.zip` included in this repo.
2. Connect the ESP32‑C3 board to PC via USB.
3. Put the board into boot mode (Boot/EN — see your board docs).
4. In the tool select **ESP32‑C3** chip.
5. Specify binaries and addresses. Typical config:
   - `meteo_station.ino.bootloader.bin` → `0x0000`
   - `meteo_station.ino.partitions.bin` → `0x8000`
   - `meteo_station.ino.bin` → `0x10000`
6. Choose your board’s COM port.
7. Click **START** and wait until flashing completes.

Always verify exact addresses and file set against the specific release — they may differ.

#### 1.2. Flashing the combined binary

If the release provides `meteo_station.ino.merged.bin`, it already contains bootloader, partitions, and main firmware combined. Then flash **one file** starting at `0x0`:

- Ensure you use the `meteo_station.ino.merged.bin` from the matching release.

---

## Communication protocol (brief)

### BLE JSON messages

- **Weather data** — periodic notifications with temperature, humidity, pressure, altitude.
- **Wi‑Fi status**:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
{"type":"wifi_status","status":"error","reason":"wrong_password"}
```

- **Mode change**:

```json
{"type":"mode_changed","mode":"BLE"}
{"type":"mode_changed","mode":"STA"}
```

### HTTP interface

- Web page: `http://<ip_address>/`
- Metrics: `http://<ip_address>/metrics` (JSON)

---

## Web Interface

The station provides a comprehensive web interface with the following features:

### Main Page

- Display of current measurements (temperature, humidity, pressure, altitude)
- Status indicators (Online, Wi‑Fi Connected)

### Menu Buttons

- **📊 Charts** — Real‑time sensor charts (updates every second)
- **📈 Statistics** — Historical data view with configurable time range (1 hour to all data)
- **🌐 Language** — Interface language selection (4 languages available)
- **⚙️ Units** — Measurement units and calibration settings
- **📡 BT Mode** — Reset to BLE mode (requires restart)

### Language Setting

The web interface supports 4 languages:

- 🇷🇺 Русский (Russian)
- 🇬🇧 English
- 🇺🇦 Українська (Ukrainian)
- 🇩🇪 Deutsch (German)

The language setting is saved and persists after restart.

### Measurement Units

In the **Units** menu you can choose:

- **Temperature**: °C or °F
- **Pressure**: hPa, kPa or mmHg
- **Altitude**: meters or feet

### Calibration Values

If sensors show systematic deviations, you can enter correction values in the same menu:

- Temperature (step 0.1 °C)
- Humidity (step 0.1 %)
- Pressure (step 0.1 hPa)
- Altitude (step 0.1 m)

Corrections are applied to all displayed data and saved.

### BT Mode Button

The **📡 BT Mode** button resets the station to BLE operating mode:

- After confirmation, the station reboots
- After reboot, the station operates in BLE mode
- Wi‑Fi settings are preserved (not deleted)

This is useful when you want to reconfigure the station via the Android app.

### Hardware Button: Factory Reset

The board has a **BOOT button** (GPIO 9). It allows a complete factory reset:

- Hold the BOOT button for **3 seconds**
- The station clears all settings (Wi‑Fi, language, units, calibration values)
- After clearing, the station reboots
- After reboot, the station operates in BLE mode (like after first flash)

Factory reset is useful when you want to completely clear all settings.

### Firmware Update via Android App

The new version of the Android app allows **flashing and updating firmware** directly via a **USB‑OTG cable**:

- Connect the station via USB‑OTG to the Android device
- The app automatically detects the station
- Select the desired firmware version
- The flashing process happens directly through the app

This doesn't replace BLE configuration, but provides a convenient way to update firmware without a PC.

---

## Logs and performance

- Excess Serial debug output is disabled.
- Logs keep only user‑relevant data:
  - weather JSON;
  - command responses;
  - key status messages.

This reduces port load and simplifies analysis.

---

## Android app

A separate Android app is used with the station:

- Connects to the station via BLE.
- Configures Wi‑Fi network for STA mode.
- Switches BLE / Wi‑Fi modes.
- Displays current weather data.

The app is distributed separately (via Google Play) and **is not built or published in source** in this repository.

---

## License

Firmware and related binaries in this repository are provided exclusively for personal, non‑commercial use.

Permitted:

- use the firmware on your own devices for personal purposes;
- modify the firmware for personal use;
- share unmodified binaries with other users, also for personal use only.

Prohibited without prior written permission of the right holder:

- use the firmware or its modifications in commercial products and services;
- sell the firmware (separately or as part of devices);
- include the firmware in any paid solutions or offerings.

All rights to the firmware and related materials remain with the project author.

---

## Русский (оригинал)

Метеостанция на базе **ESP32‑C3** с датчиком **AHT20** + **BMP280** и фирменным Android‑приложением.

Этот репозиторий предназначен **для распространения готовых прошивок (бинарных файлов)** и описания возможностей устройства. Исходный код Android‑приложения и детали его сборки здесь не публикуются.

---

## Возможности станции

- Поддержка двух режимов работы:
  - **BLE (Bluetooth Low Energy)** — начальная настройка и получение метеоданных по BLE.
  - **Wi‑Fi STA (клиент)** — плата подключается к вашему роутеру и отдаёт данные по HTTP.
- Автоопределение датчика на I2C‑шине (**AHT20**, **BMP280**).
- Передача метеоданных в формате **JSON**:
  - по BLE (уведомления характеристики);
  - по HTTP (`/metrics`) в Wi‑Fi режиме.
- Встроенный веб‑сервер:
  - веб‑страница с интерфейсом по адресу `http://<ip_адрес>/`;
  - JSON‑метрики по адресу `http://<ip_адрес>/metrics`.
- Служебные JSON‑сообщения по BLE:
  - статус Wi‑Fi (`wifi_status`);
  - смена режима работы (`mode_changed`).

Android‑приложение подключается к станции, показывает текущие измерения и позволяет настраивать Wi‑Fi **без изменения прошивки**.

---

## Режимы работы

### BLE (Bluetooth Low Energy)

- Станция после прошивки стартует в **BLE‑режиме**.
- ESP32‑C3 рекламируется с именем, например, `MeteoStation`.
- Телефон (через фирменное приложение) находит устройство, подключается по BLE и:
  - получает текущие метеоданные в JSON;
  - может отправлять настройки Wi‑Fi сети на станцию.

### Wi‑Fi STA (клиент)

- После получения настроек Wi‑Fi от телефона станция переключается в режим **Wi‑Fi STA**.
- Подключается к вашему роутеру и получает IP‑адрес.
- По этому IP доступны:
  - веб‑интерфейс: `http://<ip_адрес>/`;
  - метрики в формате JSON: `http://<ip_адрес>/metrics`.
- Статус подключения и IP‑адрес отправляются обратно в приложение по BLE сообщением:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
```

При ошибке подключения отправляется сообщение с `status="error"` и `reason`.

---

## Подключение датчиков AHT20 + BMP280

Оба датчика работают по шине **I2C** и могут быть подключены параллельно (общая шина SDA/SCL).

### Пины ESP32‑C3 (LOLIN C3 Mini)

Согласно прошивке `meteo_station.ino`:

- `SDA_PIN = 6`
- `SCL_PIN = 7`

То есть нужно подключить:

- **AHT20**:
  - `VCC` → 3.3V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA)
  - `SCL` → GPIO **7** (SCL)

- **BMP280**:
  - `VCC` → 3.3V ESP32‑C3
  - `GND` → GND ESP32‑C3
  - `SDA` → GPIO **6** (SDA) — к той же линии, что и AHT20
  - `SCL` → GPIO **7** (SCL) — к той же линии, что и AHT20

Питание датчиков — **строго 3.3V** (как у ESP32‑C3). Оба датчика висят на одной I2C‑шине, их адреса задаются внутри прошивки:

- `AHT20` — `0x38`
- `BMP280` — `0x76` или `0x77` (автоопределяется прошивкой)

Можно использовать как два отдельных модуля AHT20 и BMP280, так и **комбинированный модуль "2‑в‑1" (AHT20 + BMP280)**.
Во втором случае подключение точно такое же — всего **4 провода** к ESP32‑C3: 3.3V, GND, SDA (GPIO 6) и SCL (GPIO 7).

---

## Структура релизов

В разделе **Releases** (или в папке с бинарниками) выкладываются готовые файлы прошивки.
Для текущей платы ESP32‑C3 (профиль `esp32.esp32.lolin_c3_mini`) используются файлы:

- `meteo_station.ino.bootloader.bin`  — загрузчик (bootloader)
- `meteo_station.ino.partitions.bin`  — таблица разделов (partitions)
- `meteo_station.ino.bin`              — основная прошивка (приложение)
- `meteo_station.ino.merged.bin`       — **комбинированный бинарник**, включающий все вышеперечисленные части

В релизах можно публиковать как отдельные файлы, так и только комбинированный `meteo_station.ino.merged.bin` — в зависимости от того, как удобнее пользователям.

---

## Прошивка готового бинарника

> ВНИМАНИЕ: все действия по прошивке вы выполняете на свой страх и риск. Убедитесь, что выбираете правильный порт и модель платы.

Для прошивки ESP32‑C3 вы можете использовать:

- Официальную утилиту **ESP32 Flash Download Tool** (GUI от Espressif).
- Или консольную утилиту **esptool.py**.

### Вариант 1: ESP32 Flash Download Tool (GUI)

1. Скачайте "ESP32 Flash Download Tool" с сайта Espressif:

   <https://docs.espressif.com/projects/esp-test-tools/en/latest/esp32/production_stage/tools/flash_download_tool.html>

   Или используйте архив `flash_download_tool.zip`, приложенный в этом репозитории.
2. Подключите плату ESP32‑C3 к ПК по USB.
3. Переведите плату в режим загрузки (Boot/EN — см. документацию на вашу плату).
4. В утилите выберите нужный чип (**ESP32‑C3**).
5. Укажите бинарные файлы и адреса, используя файлы из релиза. Для типичной конфигурации можно использовать:
   - `meteo_station.ino.bootloader.bin` → `0x0000`
   - `meteo_station.ino.partitions.bin` → `0x8000`
   - `meteo_station.ino.bin` → `0x10000`
6. Выберите COM‑порт вашей платы.
7. Нажмите **START** и дождитесь окончания прошивки.

Точные адреса и состав бинарников обязательно сверяйте с описанием конкретного релиза — они могут отличаться.



#### 1.2. Прошивка комбинированного бинарника

Если в релизе есть файл `meteo_station.ino.merged.bin`, он уже содержит bootloader, partitions и основную прошивку, собранные вместе.
В этом случае достаточно прошить **один файл** с адреса `0x0`:

- Убедитесь, что используете именно `meteo_station.ino.merged.bin` из соответствующего релиза.

---

## Протокол обмена (кратко)

### BLE JSON сообщения

- **Метеоданные** — периодические уведомления с температурой, влажностью, давлением, высотой.
- **Статус Wi‑Fi**:

```json
{"type":"wifi_status","status":"ok","ip":"192.168.x.x"}
{"type":"wifi_status","status":"error","reason":"wrong_password"}
```

- **Смена режима**:

```json
{"type":"mode_changed","mode":"BLE"}
{"type":"mode_changed","mode":"STA"}
```

### HTTP интерфейс

- Веб‑страница: `http://<ip_адрес>/`
- Метрики: `http://<ip_адрес>/metrics` (JSON)

---

## Веб‑интерфейс

Станция предоставляет расширенный веб‑интерфейс со следующими функциями:

### Главная страница

- Отображение текущих измерений (температура, влажность, давление, высота)
- Индикаторы статуса (Онлайн, Wi‑Fi подключено)

### Меню кнопок

- **📊 Графики** — графики в реальном времени (обновление каждую секунду)
- **📈 Статистика** — просмотр истории с настраиваемым периодом (от 1 часа до всех данных)
- **🌐 Язык** — выбор языка интерфейса (4 языка)
- **⚙️ Единицы** — настройка единиц измерения и корректировка
- **📡 BT режим** — возврат в BLE‑режим (требуется перезагрузка)

### Настройка языка

Веб‑интерфейс поддерживает 4 языка:

- 🇷🇺 Русский
- 🇬🇧 English
- 🇺🇦 Українська
- 🇩🇪 Deutsch

Настройка языка сохраняется и восстанавливается после перезагрузки.

### Единицы измерения

В меню **Единицы** можно выбрать:

- **Температура**: °C или °F
- **Давление**: hPa, kPa или mmHg
- **Высота**: метры или футы

### Корректировка показаний

Если датчики имеют систематические погрешности, можно ввести поправки:

- Температура (шаг 0,1 °C)
- Влажность (шаг 0,1 %)
- Давление (шаг 0,1 hPa)
- Высота (шаг 0,1 м)

Поправки применяются ко всем отображаемым данным и сохраняются.

### Кнопка BT режим

Кнопка **📡 BT режим** переводит станцию в BLE‑режим:

- После подтверждения станция перезагрузится
- После перезагрузки станция работает в BLE‑режиме
- Настройки Wi‑Fi сохраняются (не удаляются)

Это полезно, если нужно перенастроить станцию через Android‑приложение.

### Аппаратная кнопка: Сброс к заводским настройкам

На плате расположена кнопка **BOOT** (GPIO 9). Она позволяет выполнить полный сброс:

- Удерживайте кнопку BOOT **3 секунды**
- Станция удалит все настройки (Wi‑Fi, язык, единицы, корректировки)
- После удаления станция перезагрузится
- После перезагрузки станция работает в BLE‑режиме (как после первой прошивки)

Сброс полезен, если нужно полностью очистить все настройки.

### Обновление прошивки через Android‑приложение

Новая версия Android‑приложения позволяет **прошивать и обновлять прошивку** напрямую через **USB‑OTG‑кабель**:

- Подключите станцию через USB‑OTG к Android‑устройству
- Приложение автоматически определит станцию
- Выберите нужную версию прошивки
- Процесс прошивки происходит напрямую через приложение

Это не заменяет BLE‑настройку, а предоставляет удобный способ обновления прошивки без ПК.

---

## Логи и производительность

- В прошивке отключён лишний отладочный вывод в Serial.
- В логах остаются только данные, полезные для пользователя:
  - метео‑JSON;
  - ответы на команды;
  - ключевые сообщения о состоянии.

Это уменьшает нагрузку на порт и упрощает анализ работы станции.

---

## Android‑приложение

Для работы с метеостанцией используется отдельное Android‑приложение:

- Подключение к станции по BLE.
- Настройка Wi‑Fi сети для режима STA.
- Переключение режимов BLE / Wi‑Fi.
- Отображение текущих метеоданных.

Приложение распространяется отдельно (через Google Play) и в этом репозитории **не собирается и не публикуется в исходниках**.

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
