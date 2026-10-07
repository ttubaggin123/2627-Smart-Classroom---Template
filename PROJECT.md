# Projektbeschreibung – ausfüllbare Vorlage

> Diese Datei beschreibt dauerhaft Projektidee, Anforderungen auf Überblicksebene und Architektur. Sie enthält **keinen laufenden Projektstatus**. Aufgaben, Verantwortlichkeiten und Fortschritt werden ausschließlich über GitHub Issues und das GitHub Project verwaltet.

## 1. Gruppenname

**Gruppenname:** …

## 2. Teammitglieder und GitHub-Namen

| Name | GitHub-Benutzername |
|---|---|
| … | @… |
| … | @… |

## 3. Gruppenthema

**Gewähltes Thema aus [GROUPS.md](GROUPS.md):** 6. Energie- und Heizungsüberwachung

Das Thema wird als Erkennung von Wärmeverschwendung im Klassenraum interpretiert. Im Mittelpunkt steht der Zustand „Heizen bei offenem Fenster“, der in der Heizperiode häufig auftritt und viel Energie kostet, ohne dass es im Raum bemerkt wird. Zusätzlich werden Überheizung und der Lüftungsvorgang bewertet. Der Heizungszustand wird ausschließlich berührungssicher über die Oberflächentemperatur des Heizkörpers bestimmt, ohne Eingriff in die Schulheizung. Ergebnis ist ein Betriebszustand, eine Warnung mit Handlungsempfehlung und die Kennzahl „vermeidbarer Wärmeverlust pro Tag“.

## 4. Problemstellung

In der Heizperiode werden Klassenräume häufig gleichzeitig beheizt und gelüftet. Besonders dauerhaft gekippte Fenster bei aufgedrehtem Heizkörper führen zu hohem, vermeidbarem Wärmeverlust, ohne dass dies im Raum jemandem auffällt. Zusätzlich wird Überheizung (Raumtemperatur über 23 °C bei laufender Heizung) oft nicht bemerkt.

Das System untersucht daher die Fragestellung:

> **Wie oft und wie lange wird im Klassenraum bei offenem Fenster geheizt, wie viel Wärmeenergie geht dabei ungefähr verloren, und wie kann das System rechtzeitig eine begründete Warnung mit Handlungsempfehlung ausgeben?**

Dazu werden Raumtemperatur, Luftfeuchte, Fensterzustand und Heizungszustand lokal auf einem ESP32 zusammengeführt. Daraus entstehen ein Betriebszustand (z. B. `heating-window-open`), eine Warnung und die Kennzahl „vermeidbarer Wärmeverlust pro Tag in kWh“.

## 5. Sensoren

Die konkrete Auswahl erfolgt in M1/M2 und muss begründet werden.

| Messgröße | geplanter Sensor / Prinzip | Zweck | Auswahlbegründung |
|---|---|---|---|
| Raumtemperatur, relative Luftfeuchte | Sensirion SHT31, digital über I²C (3,3 V) | Grundlage für Überheizungs-Erkennung, Temperatursturz-Erkennung und Verlustberechnung | Hohe Genauigkeit (typ. ±0,3 K / ±2 % rF), werkseitig kalibriert, einfache I²C-Anbindung, gut unterstützt in ESPHome und Arduino |
| Heizkörper-Oberflächentemperatur | DS18B20 (wasserdicht), 1-Wire, mit Kabelbinder oder Klettband ohne Klebeseite am Vorlaufrohr angelegt | Ableitung „Heizung aktiv“ aus der Temperaturdifferenz Heizkörper – Raum | Berührungssicher, keine Veränderung der Heizungsanlage, kein Bohren oder Kleben, wasserdicht, digital und störunempfindlich |
| Fensterzustand (offen/zu) | Funk-Fensterkontakt 433 MHz (Reed-Prinzip, batteriebetrieben, Codierung EV1527, sendet „offen“ und „zu“) + Empfänger RXB6 am ESP32 | Direkte Erkennung, ob das Fenster geöffnet ist | Keine Verkabelung am Fenster nötig, Befestigung mit Klemmhalter ohne Bohren und Kleben, günstig, einfache Auswertung |
| Fensterzustand (Plausibilisierung) | Softwaresensor: Temperaturgradient aus dem SHT31 | Zweite, unabhängige Erkennung eines Öffnungsvorgangs | Redundanz bei leerer Batterie, Funkstörung oder verrutschtem Kontakt, keine zusätzliche Hardware |
| Außentemperatur | Keine eigene Hardware, Wert kommt über MQTT aus Home Assistant (Wetterintegration) | Temperaturdifferenz innen/außen für die Verlustberechnung | Kein Außensensor montierbar, da keine Eingriffe am Gebäude erlaubt sind |

## 6. Aktoren

| Aktor / Ausgabe | Aufgabe | Auslöser | Sicherheitsgrenzen |
|---|---|---|---|
| RGB-Status-LED am Gerät | Ampelanzeige im Raum: grün = normal, gelb = Lüften oder Überheizung, rot = Heizen bei offenem Fenster | Änderung des Betriebszustands | Nur 3,3 V aus dem ESP32, Vorwiderstände, Strom pro Farbe unter 10 mA |
| TTS-Hinweis über die gemeinsame Schnittstelle (`smartclassroom/shared/tts/request`) | Einmalige gesprochene Erinnerung „Bitte das Fenster schließen oder die Heizung zudrehen.“ | Warnung aktiv seit mindestens 10 min | Nur über die freigegebene Home-Assistant-Automatisierung von Gruppe 7, `priority: low`, `volume` höchstens 0.3, höchstens eine Anfrage pro 30 min |
| Benachrichtigung in Home Assistant | Warnung mit Handlungsempfehlung („Fenster schließen oder Thermostat zudrehen, stoßlüften statt kippen“) | Warnung aktiv (mindestens 5 min Heizen bei offenem Fenster) | Reine Software-Ausgabe |

Es werden **keine Aktoren verwendet, die in die Heizung oder das Gebäude eingreifen** (z. B. Stellantriebe am Thermostatventil oder Relais). Eingriffe in die Schulheizung und Arbeiten an 230 V sind ausdrücklich verboten. Das System gibt daher nur Hinweise und Empfehlungen, die Handlung erfolgt durch Personen im Raum.

## 7. Gewählter Softwarestack

- **ESPHome-Prototyp:** ESPHome (aktuelle stabile Version, bei Projektstart festgelegt) mit den Komponenten `sht3xd`, `dallas_temp`, `remote_receiver` (rc_switch), `mqtt` mit eigener Topic-Struktur nach `docs/mqtt.md`, Template-Sensoren und einem Intervall-Lambda für die Auswertelogik. Ziel ist die schnelle Überprüfung von Hardware, Funkcodes, Schwellwerten und Datenfluss.
- **Verpflichtende Arduino-/FreeRTOS-Lösung:** Arduino-Framework auf dem ESP32 (Arduino-ESP32-Core 3.x), entwickelt mit PlatformIO. Aufteilung in FreeRTOS-Tasks:
  - `taskSensors`: alle 30 s SHT31 und DS18B20 lesen, Plausibilitätsprüfung
  - `taskRadio`: 433-MHz-Empfang, Fensterzustand setzen
  - `taskLogic`: Filterung, Zustandsbildung, Verlustberechnung, Status-LED
  - `taskMqtt`: Verbindung mit Last Will, Wiederverbindung ohne Blockieren, Publish, Subscribe, Discovery
  - Datenaustausch über FreeRTOS-Queues, gemeinsamer Zustand mit Mutex geschützt
- **Bibliotheken und Versionen:** (bei Projektstart festlegen und in `platformio.ini` fixieren)
  - Adafruit SHT31 Library 2.x
  - OneWire 2.3.x, DallasTemperature 3.9.x oder neuer
  - rc-switch 2.6.x
  - PubSubClient 2.8
  - ArduinoJson 7.x
- **Home Assistant / Grafana:** MQTT-Broker und Home Assistant auf dem zentralen Raspberry Pi, Einbindung über MQTT Discovery, Automationen für Außentemperatur, Benachrichtigung und TTS-Anfrage, Speicherung der Verläufe, Grafana-Dashboard mit Temperaturverlauf inklusive markierter Fenster-offen-Phasen, Zustandszeitleiste und vermeidbarem Wärmeverlust pro Tag.

**Übergang ESPHome → Arduino/FreeRTOS:**

1. Mit ESPHome werden Verdrahtung, Messwerte, Funkcodes und Schwellwerte validiert.
2. Die Auswertelogik aus dem ESPHome-Lambda wird als eigene C++-Klassen übernommen und mit denselben Testdaten verglichen.
3. Sensorabfrage und MQTT-Kommunikation werden schrittweise in eigene FreeRTOS-Tasks überführt.
4. `group_id`, `device_id`, `object_id`s, Topics, Payloads und Geräte-Metadaten bleiben gemäß `docs/mqtt.md` identisch, damit Home Assistant und Grafana ohne Änderung weiterlaufen.
5. Abschließend werden beide Lösungen über mindestens einen Tag verglichen.

## 8. Lokale Datenverarbeitung

Die gesamte Bewertung erfolgt direkt auf dem ESP32. Home Assistant empfängt fertige Zustände und Kennzahlen, nicht nur Rohdaten.

| Eingangsdaten | Verfahren | abgeleiteter Zustand | Begründung |
|---|---|---|---|
| Raum- und Heizkörpertemperatur | Plausibilitätsprüfung (Wertebereich, NaN, Sensor-Timeout) | `quality`: `valid`, `invalid` oder `stale` | Fehlerhafte Werte dürfen keine Fehlwarnung auslösen und müssen sichtbar markiert werden |
| Raum- und Heizkörpertemperatur | Gleitender Mittelwert über 5 Messungen | geglättete Temperaturen | Unterdrückt Messrauschen und kurzzeitige Ausreißer |
| Differenz Heizkörper – Raum | Schwellwert mit Hysterese (ein über 10 K, aus unter 6 K) | `heating-active` (true/false) | Heizungsstatus ohne Eingriff in die Anlage, Hysterese verhindert Flattern |
| Raumtemperatur | Gradient über 2 min (Abfall über 0,5 K) | `temperature-drop` (true/false) | Zweite, unabhängige Fenstererkennung |
| Funkcode Fensterkontakt | Codezuordnung, Entprellung | `window-open` (true/false) | Direkte, eindeutige Fenstererkennung |
| Fenster, Heizung, Zeit | Zeitzähler mit Verzögerung 5 min | `heating-window-warning` (true/false) | Kurzes Stoßlüften ist erwünscht und soll keine Warnung auslösen |
| Raum- und Außentemperatur, Fenster, Heizung | Integration: Q = V · 0,34 Wh/(m³·K) · ΔT · n · Δt, Tagesreset um Mitternacht | `avoidable-heat-loss` (kWh) | Macht die Energieverschwendung als vergleichbare Kennzahl sichtbar |
| alle obigen Zustände | Zustandsautomat mit Priorität | `operating-state`: `heating-off`, `heating`, `ventilating`, `overheating`, `heating-window-open` | Ein verständlicher Gesamtzustand für LED, Home Assistant und Grafana |

Der Luftwechsel n ist eine dokumentierte Annahme (Startwert 2 pro Stunde bei offenem Fenster) und wird durch eigene Messungen des Temperaturabfalls überprüft.

## 9. Datenfluss

Beispiel: Sensor → ESP32 → Prüfung → lokale Verarbeitung → Zustandsbildung → MQTT → Home Assistant → Datenspeicherung → Grafana

**Unser Datenfluss:**

SHT31, DS18B20 und 433-MHz-Fensterkontakt → ESP32 → Plausibilitätsprüfung → Filterung (gleitender Mittelwert, Hysterese, Gradient) → Zustandsbildung und Verlustberechnung → Status-LED und MQTT → Home Assistant (Anzeige, Automationen, Benachrichtigung, TTS-Anfrage) → Datenspeicherung → Grafana-Dashboard

Rückkanal: Home Assistant (Wetterintegration) → MQTT `command/outdoor-temperature` → ESP32

## 10. MQTT-Schnittstellen

Alle Topics, Payloads, Availability und Discovery folgen dem verbindlichen [MQTT-Standard](docs/mqtt.md).

- **group_id:** `group-06`
- **device_id:** `heating-node`
- **node_id:** `group-06-heating-node`
- **Topic-Basis:** `smartclassroom/group-06/heating-node`

**Messwerte und Zustände** (`.../state/<object_id>`, retained, JSON mit `value`, `unit`, `quality`, `timestamp`):

| object_id | Datentyp | Einheit | quality / Wertebereich | Intervall / Ereignis |
|---|---|---|---|---|
| `room-temperature` | Zahl | °C | −10 bis 50, sonst `invalid` | alle 30 s |
| `room-humidity` | Zahl | % | 0 bis 100, sonst `invalid` | alle 30 s |
| `radiator-temperature` | Zahl | °C | −10 bis 90, sonst `invalid` | alle 30 s |
| `window-open` | Boolean | `""` | `stale`, wenn der Kontakt länger als 24 h nichts gesendet hat | bei Änderung |
| `temperature-drop` | Boolean | `""` | true/false | bei Änderung |
| `heating-active` | Boolean | `""` | true/false | bei Änderung |
| `operating-state` | String | `""` | `heating-off`, `heating`, `ventilating`, `overheating`, `heating-window-open` | bei Änderung |
| `heating-window-warning` | Boolean | `""` | true/false | bei Änderung |
| `avoidable-heat-loss` | Zahl | kWh | ab 0, Tagesreset um 00:00 | alle 60 s |

Temperaturen werden alle 30 s gesendet, weil sich die Raumtemperatur langsam ändert und der Temperatursturz trotzdem innerhalb von 2 min erkannt wird. Zustände werden nur bei Änderung gesendet, um unnötigen MQTT-Verkehr zu vermeiden.

**Availability und Betriebszustand:**

- `.../availability`: `online` / `offline` (Last Will, QoS 1, retained)
- `.../status`: JSON mit `state` (`starting`, `running`, `degraded`, `error`), `uptime_s`, `firmware`, `rssi_dbm`, `error`. `degraded` wird gesetzt, wenn ein Sensor ungültige Werte liefert.

**Befehle** (`.../command/<object_id>`, nicht retained, JSON mit `value` und `request_id`):

| object_id | Datentyp | Einheit | Wertebereich | Wirkung |
|---|---|---|---|---|
| `outdoor-temperature` | Zahl | °C | −30 bis 45 | Außentemperatur für die Verlustberechnung, alle 10 min von Home Assistant |
| `reset-heat-loss` | Boolean | `""` | true | Tageszähler manuell zurücksetzen |

**TTS-Anfrage** an `smartclassroom/shared/tts/request`:

```json
{
  "source": "group-06",
  "message": "Bitte das Fenster schließen oder die Heizung zudrehen.",
  "priority": "low",
  "volume": 0.3
}
```

**Discovery:** `homeassistant/<component>/group-06-heating-node/<object_id>/config`, mit `unique_id` nach dem Schema `group-06-heating-node-<object_id>`, beim Start und nach jeder Wiederverbindung retained.

## 11. Abgrenzung

**Unser Teilsystem übernimmt:**

- Erfassung von Raumtemperatur, Luftfeuchte, Heizkörpertemperatur und Fensterzustand in einem Klassenraum
- Lokale Auswertung, Zustandsbildung, Warnung und Verlust-Kennzahl auf dem ESP32
- Status-LED am Gerät
- Home-Assistant-Automationen für Außentemperatur, Benachrichtigung und TTS-Anfrage sowie ein eigenes Grafana-Dashboard

**Nicht zum Projektumfang gehören:**

- Steuerung oder Veränderung der Heizung (Thermostate, Ventile, Heizkurven)
- Messung des elektrischen Energieverbrauchs an fest installierten 230-V-Anlagen
- Erfassung von Personen, Präsenz oder Audio
- Montage am Gebäude durch Bohren oder Kleben

**Andere Gruppen und zentrale Dienste:**

- Betrieb von Raspberry Pi, MQTT-Broker, Home Assistant, Datenspeicherung und Grafana durch die Server-Gruppe
- Ausgabe der TTS-Hinweise über den Lautsprecher durch Gruppe 7
- WLAN und Netzwerk durch die Schule bzw. die Server-Gruppe

## 12. Sicherheits- und Datenschutzbetrachtung

**Elektrische Sicherheit / 230 V:** Das Gerät wird ausschließlich über ein CE-geprüftes USB-Netzteil versorgt. Es werden keine Arbeiten an 230 V durchgeführt, keine Geräte geöffnet und keine Stromzangen im Verteiler verwendet. Alle eigenen Schaltungen arbeiten mit 3,3 V bzw. 5 V.

**Schulhardware und Infrastruktur:** Die Heizung und das Gebäude werden nicht verändert. Der Heizkörperfühler wird nur mit Kabelbinder oder Klettband ohne Klebeseite außen angelegt, der Fensterkontakt mit Klemmhaltern befestigt. Alle Befestigungen sind rückstandsfrei entfernbar und werden vorab mit der Lehrperson abgestimmt. Kabel werden ohne Stolpergefahr verlegt.

**Funk (433 MHz):** Das Protokoll ist unverschlüsselt und könnte gestört oder nachgeahmt werden. Das Risiko ist gering, da nur Hinweise entstehen und nichts gesteuert wird. Der Fensterzustand wird zusätzlich über den Temperaturgradienten plausibilisiert.

**Audio, Bluetooth/ESPresense:** Werden nicht verwendet. Es erfolgt keine Erfassung von Sprache, Geräten oder Personen.

**TTS:** Anfragen nur über die freigegebene Schnittstelle, mit sachlichem Text ohne Personenbezug, `priority: low`, `volume` höchstens 0.3 und höchstens einer Anfrage pro 30 min, damit der Unterricht nicht gestört wird.

**Personenbezogene Daten:** Es werden keine personenbezogenen Daten erhoben. Aus Fenster- und Heizungszustand könnte indirekt auf die Raumnutzung geschlossen werden. Die Daten werden deshalb nur mit der Gerätekennung gespeichert und nicht mit Stundenplänen oder Namen verknüpft.

**Zugangsdaten:** WLAN-, MQTT- und OTA-Zugangsdaten liegen in `secrets.yaml` bzw. `secrets.h`, die über `.gitignore` nicht ins Repository gelangen. Keine Payload enthält Geheimnisse. OTA-Updates sind passwortgeschützt.

**Verbleibende Risiken:**

- Fehlwarnungen bei ungünstiger Sensorplatzierung (z. B. Sonne auf dem Gehäuse, Gerät direkt über dem Heizkörper)
- Die Verlust-Kennzahl ist eine Abschätzung mit angenommenem Luftwechsel
- Batterieausfall des Fensterkontakts, abgefangen durch Plausibilisierung und `quality: stale`
- Unverschlüsselte MQTT-Verbindung, falls der zentrale Broker kein TLS anbietet

## 13. Abnahmekriterien

Die IDs werden in [docs/requirements.md](docs/requirements.md) mit den zugehörigen Tests aus [docs/testing.md](docs/testing.md) eingetragen.

- [ ] **REQ-F-01:** Raum- und Heizkörpertemperatur werden mindestens alle 30 s mit gültigem `quality`-Feld veröffentlicht. Die Raumtemperatur weicht höchstens ±0,5 K von einem Referenzthermometer ab.
- [ ] **REQ-F-02:** Eine Fensteröffnung wird in 10 von 10 Versuchen innerhalb von 5 s erkannt und als `window-open` veröffentlicht.
- [ ] **REQ-F-03:** Der Heizungszustand wird bei auf- und zugedrehtem Heizkörper korrekt als `heating-active` erkannt, ohne mehrfaches Umschalten im Übergangsbereich.
- [ ] **REQ-F-04:** Nach 5 min (±30 s) Heizen bei offenem Fenster wird `heating-window-warning` aktiv. Bei Stoßlüften unter 5 min entsteht keine Warnung.
- [ ] **REQ-F-05:** `avoidable-heat-loss` stimmt mit einer Handrechnung aus den gespeicherten Daten auf ±5 % überein und wird um Mitternacht zurückgesetzt.
- [ ] **REQ-F-06:** Der Betriebszustand wird in Home Assistant angezeigt, bei Warnung erscheint eine Benachrichtigung mit Handlungsempfehlung.
- [ ] **REQ-F-07:** Das Grafana-Dashboard zeigt Temperaturverlauf mit Fenster-offen-Phasen, Zustandszeitleiste und Verlust pro Tag.
- [ ] **REQ-NF-01:** Alle Topics, Payloads und Discovery-Konfigurationen entsprechen `docs/mqtt.md` (Prüfung mit MQTT Explorer).
- [ ] **REQ-NF-02:** Bei Stromausfall wird `offline` über Last Will gemeldet. Nach MQTT-Ausfall verbindet sich das Gerät selbstständig wieder und veröffentlicht Availability, Status, Zustände und Discovery erneut.
- [ ] **REQ-NF-03:** Ein abgesteckter Sensor führt zu `quality: invalid` und `state: degraded`, aber zu keiner Fehlwarnung.
- [ ] **REQ-NF-04:** Die Arduino-/FreeRTOS-Lösung liefert über mindestens einen Tag dieselben Zustände wie der ESPHome-Prototyp.
- [ ] **REQ-NF-05:** Das System läuft 24 h ohne Neustart.
- [ ] **REQ-NF-06:** Im Repository sind keine Zugangsdaten enthalten.
