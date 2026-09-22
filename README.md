# 4-Zoll ESP32-S3 Raumdisplay – Anleitung

Modernes Raumdisplay für **Home Assistant** auf Basis des **Guition ESP32-S3-4848S040**
(auch bekannt als **JC4848W540**) – 4 Zoll, 480×480 RGB-Touchscreen.
Konfiguriert mit **ESPHome + LVGL**.

> **Wichtig:** Dieser Code enthält meine persönlichen Entitäten (Sensoren, Lichter,
> Thermostat). **Jeder muss die Entitäten auf seine eigene Home-Assistant-Installation
> anpassen** – siehe Abschnitt [5. Entitäten anpassen](#5-entitäten-anpassen).

---

<img width="1920" height="1249" alt="ESP-32-S3_4Zoll_Display (1)" src="https://github.com/user-attachments/assets/42aff703-ffe6-4202-a6b9-2d3cdce128ae" />

---

## Inhalt

1. [Hardware](#1-hardware)
2. [Features im Detail](#2-features-im-detail)
3. [Voraussetzungen](#3-voraussetzungen)
4. [Erste Einrichtung & Flashen](#4-erste-einrichtung--flashen)
5. [Entitäten anpassen](#5-entitäten-anpassen)
6. [Hintergrundbild einfügen](#6-hintergrundbild-einfügen)
7. [Schriften / Icons (Netzwerkzugriff beim Kompilieren)](#7-schriften--icons)
8. [Home-Assistant-Aktionen erlauben (Lichter & Heizung schalten)](#8-home-assistant-aktionen-erlauben)
9. [Bedienung am Display](#9-bedienung-am-display)
10. [Einstellungen aus Home Assistant (Dimmung & Screensaver-Zeit)](#10-einstellungen-aus-home-assistant)
11. [Nachrichten an das Display schicken (Popup)](#11-nachrichten-an-das-display-schicken)
12. [Wie der Code aufgebaut ist](#12-wie-der-code-aufgebaut-ist)
13. [Fehlersuche](#13-fehlersuche)

---

## 1. Hardware

| Merkmal | Details |
|---|---|
| Board | Guition ESP32-S3-4848S040 / JC4848W540 |
| Display | 4″, 480×480, RGB-Panel (ST7701S) |
| Touch | GT911 (kapazitiv) |
| Speicher | 8 MB PSRAM (Octal), 16 MB Flash |
| Relais | 3 Stück (GPIO 1, 2, 40) – als „Relay 1–3" in HA schaltbar |

---

## 2. Features im Detail

### 🏠 Startseite / Screensaver
- **Große Uhrzeit** und **Datum** (deutsches Format, z. B. „So, 21.09.2026").
- **Drei Kacheln** mit Live-Werten: **Verbrauch (W)**, **Temperatur (°)**, **Batterie (%)**.
- **Dynamische Icon-Farben** je nach Wert – man sieht den Zustand auf einen Blick:

  | Kachel | Farblogik |
  |---|---|
  | ⚡ Verbrauch | grün < 300 W · gelb < 500 W · orange < 800 W · rot ≥ 800 W |
  | 🌡 Temperatur | blau < 10° · grün < 16° · gelb < 21° · orange < 25° · rot ≥ 25° |
  | 🔋 Batterie | rot < 15 % · orange < 30 % · gelb < 50 % · grün ≥ 50 % |

- **Batterie-Symbol passt sich dem Ladestand an** – das Icon zeigt in 10 %-Schritten den
  tatsächlichen Füllstand (leer → voll), nicht nur eine feste Grafik.
- **Kacheln sind anklickbar:** Verbrauch/Batterie → Energie-Seite, Temperatur → Wetter-Seite.
- **Automatischer Screensaver:** Nach der eingestellten Zeit (Standard 30 s) dimmt das
  Display, springt auf die Startseite und **blendet die grauen Kacheln samt Beschriftung
  aus** – nur Icon + Wert bleiben mittig stehen (aufgeräumte Nachttisch-Ansicht).
  Eine Berührung weckt das Display sofort wieder auf.

### 💡 Licht-Seite
- **8 Lichtschalter** mit eigenen Namen und passenden Icons.
- Schalten **real über Home Assistant** (nicht nur optisch) – Voraussetzung siehe
  [Abschnitt 8](#8-home-assistant-aktionen-erlauben).
- Der Ein/Aus-Zustand wird optisch dargestellt.

### 🌤 Wetter-Seite
- **Wetterlage** als Icon + Text (sonnig, bewölkt, Regen, Schnee, Nebel, Gewitter …).
- **Luftfeuchte** und **Windgeschwindigkeit**.
- **12-Stunden-Temperaturverlauf** als Liniendiagramm: 8 Stützpunkte, alle 90 Minuten
  ein neuer Wert (die Punkte wandern automatisch nach).

### ⚡ Energie-Seite (Tageswerte)
- **Erzeugung** (PV), **Batterieladung**, **Netzbezug** und **Hausverbrauch** in kWh.

### 🔥 Heizung-Seite
- **Wohnzimmer-Thermostat** als großer **Drehregler** (Arc).
- Zeigt **Soll- und Ist-Temperatur**, Anpassung per **+ / −** oder direkt am Regler.
- Steuert das `climate`-Gerät in HA (Temperatur setzen, Modus/Preset).

### ⚙️ System-Seite
- **Display-Helligkeit** (Slider).
- **Standby-Dimmung** (Helligkeit im Screensaver).
- **Screensaver-Timeout** (Sekunden).
- **Neustart**-Knopf für das Gerät.

### 🔔 Popup-Benachrichtigungen
- Aus HA befüllbarer Text erscheint als **Overlay-Popup** über allem.
- Verschwindet nur durch **Antippen**, neue Nachricht überschreibt die alte.
- **Stört den Screensaver nicht** – Details in [Abschnitt 11](#11-nachrichten-an-das-display-schicken).

### 🎛 Aus Home Assistant steuerbar
- **Standby-Dimmung** und **Screensaver-Timeout** als Number-Entities (auch am Gerät einstellbar).
- **3 Relais** des Boards als Schalter „Relay 1–3".
- **Hintergrundbeleuchtung** als eigener Schalter.

### 🧭 Navigation & Bedienkonzept
- **Startseite tippen** → Licht · **lang drücken** → System.
- **Kacheln tippen** → Energie / Wetter.
- **Untere Menüleiste** auf allen Unterseiten (Licht · Wetter · Energie · Heizung);
  auf der Startseite ausgeblendet.

### 🎨 Technische Feinheiten
- Zentrales **Farbschema** über `substitutions` – einmal ändern, überall wirksam.
- **Boot-sicher** aufgebaut (kein Absturz beim Start – siehe [Abschnitt 12](#12-wie-der-code-aufgebaut-ist)).
- Fehlende Sensorwerte werden als `--` dargestellt (kein Absturz bei NaN).

---

## 3. Voraussetzungen

- **Home Assistant** mit **ESPHome** (Add-on oder ESPHome Device Builder).
- USB-C-Kabel zum ersten Flashen (das Board meldet sich als serielles Gerät; unter Windows ggf. **CH340**-Treiber nötig).
- Die Datei `4zoll-esp32-display.yaml` und das Bild `hintergrund.png`.

---

## 4. Erste Einrichtung & Flashen

1. **`secrets.yaml`** in deinem ESPHome-Ordner muss WLAN-Zugangsdaten enthalten:
   ```yaml
   wifi_ssid: "DeinWLAN"
   wifi_password: "DeinWLANPasswort"
   ```

2. In `4zoll-esp32-display.yaml` die **Platzhalter** durch echte Werte ersetzen:
   ```yaml
   api:
     encryption:
       key: "xxx"          # <-- eigenen API-Key eintragen (ESPHome erzeugt einen)

   ota:
     - platform: esphome
       password: "xxx"     # <-- eigenes OTA-Passwort/-Hash

   wifi:
     ap:
       ssid: "4Zoll-ESP32-Dis Fallback"
       password: "xxx"     # <-- Fallback-AP-Passwort
   ```
   > Tipp: Einen neuen `api`-Key kannst du dir in ESPHome unter *Secrets* bzw. beim
   > Anlegen eines Geräts erzeugen lassen.

3. **Bild bereitlegen** – siehe [Abschnitt 6](#6-hintergrundbild-einfügen).

4. **Erstes Flashen per USB:** Board anschließen, in ESPHome *Install → Plug into this computer*.

5. Nach dem ersten Flashen laufen alle weiteren Updates **drahtlos (OTA)** über WLAN.

6. **Gerät in Home Assistant hinzufügen:** HA entdeckt das ESPHome-Gerät automatisch
   (*Einstellungen → Geräte & Dienste*). Ohne diesen Schritt kommen **keine Daten** auf
   das Display und die Lichtschalter reagieren nicht.

---

## 5. Entitäten anpassen

Ersetze die folgenden `entity_id`-Werte durch deine eigenen. Suche im YAML einfach
nach dem jeweiligen Eintrag (`entity_id:`) oder nach der Zeilennummer.

### Startseite / Kacheln
| Anzeige | Aktuelle Entität | Was rein soll |
|---|---|---|
| Verbrauch (W) | `sensor.total_power_kombiniert` | aktuelle Hausleistung |
| Temperatur (°) | `sensor.aussentemperatur_temperatur` | Außentemperatur |
| Batterie (%) | `sensor.victron_battery_soc_3` | Batterie-Ladestand (SoC) |

### Wetter-Seite
| Anzeige | Aktuelle Entität |
|---|---|
| Wetterlage/Icon | `weather.fuerstenwalde_spree` |
| Luftfeuchte | `sensor.aussentemperatur_luftfeuchtigkeit` |
| Wind | `sensor.fuerstenwalde_spree_windgeschwindigkeit` |
| Temperaturverlauf | `sensor.aussentemperatur_temperatur` |

### Energie-Seite (Tageswerte)
| Anzeige | Aktuelle Entität |
|---|---|
| Erzeugung | `sensor.erzeugung_taglich_kombi_pvanlagennach_ohne_abzug_einspeisung` |
| Batterieladung | `sensor.batterieladung_vr_kwh_taglich_2` |
| Netzbezug | `sensor.verbrauch_taglich_nach_abzug` |
| Hausverbrauch | `sensor.stromverbrauch_taglich` |

### Heizung
| Anzeige | Aktuelle Entität |
|---|---|
| Thermostat (Soll/Steuerung) | `climate.wandthermostat_jan_2` |
| Ist-Temperatur | `sensor.wandthermostat_jan_temperatur` |

### Licht-Seite (8 Schalter)
| Beschriftung | Aktuelle Entität |
|---|---|
| PC LED | `light.pc_led_pv_led` |
| Wandlicht | `light.wandlicht_h619a` |
| Küche | `light.kuchenlicht` |
| Küche Decke | `light.kuchenlicht_2` |
| Couch | `light.couchlicht` |
| Flur | `light.flurlicht` |
| Bad Spiegel | `light.bad_spiegellicht` |
| Sockelleiste | `light.sockelleisten_steckdose_1` |

> Die Beschriftungen (`text: "…"`) und Icons kannst du im YAML frei anpassen.
> Kommt ein Sensor mal nicht (NaN), zeigt das Feld `--`.

---

## 6. Hintergrundbild einfügen

Das Display nutzt ein 480×480-Bild als Hintergrund.

1. Datei **`hintergrund.png`** in denselben Ordner legen wie die YAML –
   also in **`/config/esphome/`** deiner Home-Assistant-Installation.
2. Im YAML ist es bereits eingebunden:
   ```yaml
   image:
     - file: hintergrund.png
       id: bg_img
       type: RGB565
   ```
3. **Eigenes Bild verwenden:** einfach als `hintergrund.png` (480×480 Pixel) ablegen.
   Anderer Dateiname? Dann den `file:`-Eintrag entsprechend ändern.

> Beim Screensaver wird der Hintergrund automatisch ausgeblendet
> (`bg_image_opa: TRANSP`), damit die Werte klar auf schwarzem Grund stehen.

---

## 7. Schriften / Icons

- **Montserrat** (Texte) und **Material Design Icons** werden **beim Kompilieren
  automatisch heruntergeladen** (gfonts bzw. GitHub-Webfont). Der ESPHome-Rechner
  braucht dafür **Internetzugang**.
- Umlaute und `°` funktionieren über `glyphsets: [GF_Latin_Core]`.
- Neue Icons müssen als Codepoint in die Liste `&mdi_glyphs` aufgenommen werden,
  sonst bleiben sie leer.

---

## 8. Home-Assistant-Aktionen erlauben

Damit das Display **Lichter schalten** und die **Heizung steuern** kann, muss in
Home Assistant die entsprechende Berechtigung aktiv sein:

> *Einstellungen → Geräte & Dienste → ESPHome → dein Gerät → **Konfigurieren** →
> „Dem Gerät erlauben, Home-Assistant-Aktionen auszuführen"* aktivieren.

Ohne diesen Haken reagieren die Schalter **optisch**, schalten aber real nichts.

---

## 9. Bedienung am Display

- **Startseite antippen** → Licht-Seite.
- **Lange auf die Startseite drücken** → System-Seite.
- **Kacheln antippen:** Verbrauch/Batterie → Energie, Temperatur → Wetter.
- **Untere Menüleiste** (auf allen Unterseiten): Wechsel zwischen Licht, Wetter,
  Energie, Heizung.
- **Screensaver:** Nach Ablauf der eingestellten Zeit dimmt das Display, springt auf
  die Startseite und blendet die Kacheln aus. Eine Berührung weckt es wieder auf.

---

## 10. Einstellungen aus Home Assistant

Nach dem Einbinden erscheinen beim Gerät zwei einstellbare Werte (Number-Entities):

| In HA sichtbar als | Funktion | Bereich |
|---|---|---|
| **Standby-Dimmung** | Helligkeit im Screensaver | 0–100 % |
| **Screensaver-Timeout** | Zeit bis zum Screensaver | 10–300 s |

Die Werte lassen sich **in HA** und **am Gerät** (System-Seite) verstellen.

> Hinweis: Änderst du den Wert in HA, greift das Verhalten sofort. Der Schieberegler
> auf der **System-Seite** des Displays zieht optisch aber erst nach, wenn du ihn dort
> selbst berührst (bewusst so gelöst, um das Display beim Start nicht zu stören).

---

## 11. Nachrichten an das Display schicken

Das Display hat eine Text-Entität, mit der du eine **Popup-Nachricht** anzeigen kannst.

- **Entität:** `text.<gerätename>_display_nachricht`
  (in HA unter *Entwicklerwerkzeuge → Zustände* nach `display_nachricht` suchen –
  der genaue Name hängt vom Gerätenamen ab).

**Verhalten:**
- Text setzen → Popup erscheint mittig über allem.
- Verschwindet **nur durch Antippen** des Displays.
- Eine neue Nachricht **überschreibt** die alte.
- **Text leeren** (`""`) → Popup wird ausgeblendet.
- Der **Screensaver bleibt aktiv** – das Popup verhindert das Abdunkeln nicht und ist
  nach dem Aufwecken weiterhin sichtbar.

### Beispiel: Service-Aufruf (Entwicklerwerkzeuge → Aktionen)
```yaml
action: text.set_value
target:
  entity_id: text.4zoll_esp32_display_display_nachricht
data:
  value: "Waschmaschine ist fertig!"
```

### Beispiel: Automatisierung
```yaml
alias: Türklingel aufs Display
trigger:
  - trigger: state
    entity_id: binary_sensor.tuerklingel
    to: "on"
action:
  - action: text.set_value
    target:
      entity_id: text.4zoll_esp32_display_display_nachricht
    data:
      value: "Es klingelt an der Tür 🔔"
mode: single
```

### Popup per Automatisierung wieder ausblenden (optional)
```yaml
  - delay: "00:00:30"
  - action: text.set_value
    target:
      entity_id: text.4zoll_esp32_display_display_nachricht
    data:
      value: ""
```

---

## 12. Wie der Code aufgebaut ist

- **`substitutions`** oben: Gerätename und ein zentrales Farbschema (`c_bg`, `c_card`,
  `c_text`, `c_cyan` …). Farben zentral hier ändern.
- **`globals`**: Laufzeitwerte (Screensaver-Status, Dimmwert, Timeout, Temperaturverlauf).
- **`interval: 1s`**: Screensaver-Timer + ein „Display-bereit"-Flag (`g_lvgl_ready`).
- **`number` / `text`**: die aus HA einstellbaren Regler und die Nachricht.
- **`sensor` / `text_sensor`**: holen HA-Daten und aktualisieren die Anzeige (inkl.
  dynamischer Icon-Farben).
- **`lvgl`**: die komplette Oberfläche – `top_layer` (Menüleiste + Popup) und `pages`.

### Boot-Sicherheit (wichtig beim Erweitern)
LVGL-Aktionen (`lvgl.…`) dürfen **nicht** ausgeführt werden, bevor das Display fertig
initialisiert ist – sonst startet das Gerät in eine Boot-Schleife. Deshalb:
- Komponenten mit `restore_value: true` (die beim Start automatisch feuern) rufen im
  `on_value` **nur Globals** auf, **keine** `lvgl.…`-Aktionen.
- Das Popup wird erst angezeigt, wenn das Flag `g_lvgl_ready` gesetzt ist (nach dem
  ersten Sekunden-Tick).

Wer den Code erweitert, sollte diese Regel beibehalten.

---

## 13. Fehlersuche

| Symptom | Ursache / Lösung |
|---|---|
| Display bleibt schwarz, startet immer wieder neu (Boot-Loop) | Meist eine `lvgl.…`-Aktion, die zu früh läuft. Neu eingefügten Code auf die Boot-Regel aus [Abschnitt 12](#boot-sicherheit-wichtig-beim-erweitern) prüfen. Flashen im Boot-Loop: mehrmals aus-/einstecken (ESPHome aktiviert nach einigen Fehlstarts den Safe-Mode für OTA) oder per USB. |
| Keine Daten auf dem Display | Gerät in HA hinzugefügt? WLAN/API korrekt? |
| Lichter reagieren nur optisch, schalten nicht | HA-Aktionen erlauben – siehe [Abschnitt 8](#8-home-assistant-aktionen-erlauben). |
| Icons bleiben leer | Codepoint fehlt in `&mdi_glyphs`. |
| Kompilieren bricht bei Fonts ab | Kein Internetzugang auf dem ESPHome-Rechner. |
| Umlaute/`°` fehlen | `glyphsets: [GF_Latin_Core]` bei der Schrift ergänzen. |

---

*Erstellt für die Weitergabe an andere Nutzer. Entitäten bitte an die eigene
Installation anpassen.*
