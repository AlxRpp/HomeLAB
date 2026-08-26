# Hausautomation

**Stand:** 26.08.2026 · **Status:** in Betrieb seit 26.09.2025

Alle Zahlen in diesem Dokument sind am 26.08.2026 aus der laufenden
Installation ausgelesen, nicht geschätzt. Wo etwas nicht ermittelbar war,
steht **unklar** — nicht ein plausibler Wert.

---

## 1. Zielbild

Das Haus soll auf Zustände reagieren, ohne dass jemand ein Telefon in die
Hand nimmt, und dabei so lange laufen, wie der Strom da ist — auch wenn
die Internetleitung ausfällt. Zweitens soll es eine Datenbasis erzeugen:
Sensor-Zeitreihen, aus denen sich später beantworten lässt, was wann warum
passiert ist.

Drittens ist es die erste Stufe des Jarvis-Zielbilds aus
`00-fundament.md`: Home Assistant ist nicht der Assistent, sondern die
Schicht darunter — das, was ein Modell über eine Werkzeugschnittstelle
bedienen kann. Deshalb steht in diesem Dokument neben Geräten und
Automationen auch, was ein angebundenes Modell darf und was nicht.

---

## 2. Entscheidungen

### Home Assistant als Zentrale — 26.09.2025

Die Alternative wäre gewesen, bei den Apps der Hersteller zu bleiben: Hue-App
für Licht, Shelly-App für Rollos, Home-Connect-App für die Küche, Roborock-App
für den Sauger. Fünf Apps, fünf Konten, fünf Cloud-Abhängigkeiten, und keine
Möglichkeit, ein Gerät des einen Herstellers auf ein Ereignis des anderen
reagieren zu lassen.

Home Assistant löst genau das: eine Zustandstabelle über alle Geräte hinweg,
egal über welches Protokoll sie angebunden sind. Alles Weitere — Automationen,
Historie, Sprachanbindung — setzt darauf auf und funktioniert deshalb für
jedes Gerät gleich.

Zweiter Grund, der für dieses Repo der wichtigere ist: Home Assistant ist
bereits ein MCP-Client und kann über ein Add-on auch MCP-Server sein. Die
Anbindung eines Sprachmodells ist damit Konfiguration, kein Eigenbau
(→ `00-fundament.md`, Abschnitt 7, „Core NICHT zuerst bauen").

**Umwerfen würde das:** Ein Aufwand für Pflege und Updates, der den Nutzen
übersteigt — etwa wenn Breaking Changes in Minor-Releases dazu führen, dass
nach jedem Update etwas nicht mehr läuft.

### Zigbee für Sensorik, WLAN für Aktorik — 18.11.2025

**Das ist bewusst kein „Zigbee statt WLAN".** Der Bestand sieht anders aus:
8 Zigbee-Geräte über Zigbee2MQTT, 8 weitere im Hue-Netz, aber **10 Shellys
über WLAN**. Die Aufteilung folgt einem Kriterium, nicht einer Vorliebe.

- **Batteriebetriebene Sensorik geht über Zigbee.** Ein Kontakt- oder
  Präsenzsensor soll Jahre mit einer Knopfzelle laufen. Über WLAN geht das
  nicht: Der Client muss sich beim Access Point an- und abmelden, das kostet
  ein Vielfaches der Energie eines Zigbee-Funkbursts. Alle 4 Zigbee-Geräte
  mit Batteriemeldung fallen in diese Gruppe.
- **Fest verbaute Aktorik geht über WLAN.** Die Rollo- und Lichtaktoren
  sitzen in Unterputzdosen und hängen ohnehin an 230 V. Der Energievorteil
  von Zigbee entfällt damit vollständig, und dafür gibt es zwei echte
  Vorteile: Die Geräte sind lokal per HTTP erreichbar, auch ohne Home
  Assistant, und sie belasten das Zigbee-Netz nicht.
- **Was nicht passieren darf:** WLAN-Geräte, die für ihre Grundfunktion eine
  Cloud brauchen. Shelly kann rein lokal betrieben werden — das ist das
  eigentliche Auswahlkriterium gewesen, nicht das Funkprotokoll.

**Umwerfen würde das:** Ein WLAN, das unter der Client-Zahl leidet. Bei
derzeit rund 50 Clients ist davon nichts zu sehen; ab dem Punkt, wo Rollos
verzögert reagieren, wäre Zigbee oder Thread auch für Aktorik die Antwort.

### Lokal, wo es lokal geht — 26.09.2025

Die Regel ist nicht „keine Cloud", sondern: **Was das Haus zum Funktionieren
braucht, muss ohne Internet funktionieren.**

Lokal laufen damit alle Geräte, an denen Automationen hängen: Zigbee, Hue,
Matter/Thread, Shelly. Fällt die Leitung aus, laufen Rollos, Licht, Präsenz
und Heizung weiter. Home Assistant selbst braucht kein Konto — Nabu Casa ist
nicht eingeloggt.

**Ehrlich dazugehört:** Vier Integrationen sind Cloud-gebunden — Küchengeräte
(Home Connect), Saugroboter (Roborock), Paketverfolgung und Abfuhrkalender.
Für die ersten beiden gibt es keine lokale API; das ist der Preis dafür, dass
die Geräte überhaupt eingebunden sind. Die Konsequenz ist bewusst begrenzt:
**an keinem dieser Geräte hängt eine Automation.** Fällt die Cloud aus, fehlen
Komfortfunktionen, aber nichts, was das Haus braucht.

**Umwerfen würde das:** Ein Hersteller, der eine lokale API nachliefert — dann
wandert das Gerät in die lokale Gruppe. Oder umgekehrt: ein Cloud-Dienst, der
abgeschaltet wird, und ein Gerät, das dann nur noch als Fernbedienung taugt.

### Zigbee2MQTT statt ZHA — 18.11.2025

Beide binden Zigbee-Geräte lokal an. Der Unterschied liegt darin, was zwischen
Funk und Home Assistant steht.

- **ZHA** ist eine Integration *in* Home Assistant. Die Zigbee-Anbindung ist
  damit an den HA-Prozess gebunden: HA neu starten heißt Zigbee-Netz neu
  starten, und ein HA-Ausfall nimmt die Zigbee-Anbindung mit.
- **Zigbee2MQTT** ist ein eigener Dienst, der auf einen MQTT-Broker
  veröffentlicht. Home Assistant ist damit nur ein Abonnent unter mehreren.
  Das Zigbee-Netz läuft weiter, wenn HA neu startet, und ein zweiter
  Konsument — ein eigener Dienst, ein Skript, später der Jarvis-Core — kann
  dieselben Daten mitlesen, ohne den Umweg über die HA-API.
- **Gerätekompatibilität:** Z2M unterstützt spürbar mehr Geräte und liefert
  Unterstützung für neue Modelle schneller nach als ZHA.

Der Preis ist ein Baustein mehr: Broker und Z2M sind zwei zusätzliche
Add-ons, die laufen, aktuell gehalten und gesichert werden müssen. Für das
Jarvis-Zielbild ist der Bus als eigene Schicht das wert.

**Umwerfen würde das:** Wenn MQTT als Bus nie einen zweiten Konsumenten
bekommt. Dann sind es zwei Add-ons für einen Vorteil, den niemand nutzt, und
ZHA wäre die einfachere Lösung.

### Netzwerk-Koordinator statt USB-Dongle — 18.11.2025

Der Zigbee-Koordinator (SLZB-MR1) hängt am Ethernet, nicht am USB-Port. Das
war die Voraussetzung dafür, dass Home Assistant als VM laufen kann, ohne
dass ein USB-Gerät durchgereicht werden muss — und es ist dieselbe
Entscheidung, die in `00-fundament.md`, Abschnitt 2 unter „Eine Maschine
statt zwei" steht.

Praktischer Nebeneffekt: Der Koordinator steht dort, wo die Funkabdeckung am
besten ist, nicht dort, wo der Server steht. Bei einem USB-Dongle wäre beides
derselbe Ort.

Das Gerät hat zwei Funkeinheiten — eine als Zigbee-Koordinator, eine als
Thread-Radio.

**Umwerfen würde das:** Latenz oder Aussetzer durch die Netzwerkstrecke. Bis
jetzt nicht beobachtet, aber die Strecke ist eine zusätzliche Fehlerquelle,
die ein USB-Dongle nicht hat.

### MCP für die Modellanbindung — spätestens 20.08.2026

Ein Sprachmodell soll das Haus bedienen können. Die naheliegende Variante
wäre gewesen, dem Modell direkt die Home-Assistant-REST-API zu geben. Dagegen
sprechen drei Dinge:

1. **Ein Token ist alles oder nichts.** Ein Long-Lived Access Token kann alles,
   was der Benutzer kann. MCP kennt einzelne Werkzeuge, die sich einzeln
   abschalten lassen.
2. **Das Modell müsste die API kennen.** Über MCP bekommt es benannte Werkzeuge
   mit beschriebenen Parametern statt eines Endpunkt-Schemas, das es sich aus
   dem Trainingswissen zusammenreimt.
3. **Austauschbarkeit.** MCP ist client-neutral. Welches Modell verbindet, ist
   eine Client-Entscheidung — im Server ist kein Modell hinterlegt. Damit
   bleibt der Weg zum lokalen Modell (`00-fundament.md`, Abschnitt 3, GPU)
   offen, ohne dass an der Hausseite etwas geändert werden muss.

Der MCP-Server läuft als Add-on **auf demselben Host**, nicht als externer
Dienst. Details, Rechte und Grenzen: Abschnitt 7.

**Umwerfen würde das:** Wenn Home Assistant selbst eine gleichwertige,
werkzeugbasierte Modellschnittstelle mitbringt. Dann fällt das Add-on weg,
die Architektur bleibt dieselbe.

### Nicht als Entscheidung dokumentierbar: Recorder-Aufbewahrung

Die Aufbewahrung ist **gemessen 10 Tage** (Abschnitt 5). Das ist zugleich der
Standardwert von Home Assistant. Ob er bewusst gesetzt oder nie geändert
wurde, lässt sich von außen nicht unterscheiden — die `recorder:`-Sektion
steht in `configuration.yaml` und wird von der API nicht herausgegeben.

**Bis das geklärt ist, steht die Aufbewahrung in diesem Dokument als Zustand,
nicht als Entscheidung.** Siehe Abschnitt 8.

---

## 3. Architektur

Der Datenfluss läuft nicht über einen Weg, sondern über vier parallele
Funkpfade, die sich erst in Home Assistant treffen:

```
 Zigbee-Sensorik ──Funk──▶ SLZB-MR1 ──Ethernet──▶ Zigbee2MQTT ──▶ Mosquitto ──┐
 (batteriebetrieben)       (Koordinator)            (Add-on)     (MQTT-Broker) │
                                                                               │
 Hue-Leuchten ────Funk──▶ Hue Bridge ──────────────────────LAN────────────────┤
 (eigenes Zigbee-Netz)                                                         │
                                                                               │
 Matter-Geräte ───Thread─▶ OTBR ──┐                                            │
 Matter-Geräte ───WLAN────────────┴──▶ Matter Server ───────LAN────────────────┤
                                          (Add-on)                             │
                                                                               │
 Shelly-Aktorik ──WLAN─────────────────────────────────────LAN────────────────┤
 (Unterputz, 230 V)                                                            │
                                                                               ▼
                                                                    ┌────────────────────┐
                                                                    │  Home Assistant    │
                                                                    │  (HAOS als VM)     │
                                                                    └─────────┬──────────┘
                        ┌─────────────────────────┬───────────────────────────┤
                        ▼                         ▼                           ▼
                 ┌─────────────┐          ┌──────────────┐          ┌──────────────────┐
                 │ Automationen│          │   Recorder   │          │   MCP-Server     │
                 │ (Trigger →  │          │   (SQLite)   │          │   (Add-on)       │
                 │  Bedingung  │          └──────┬───────┘          └────────┬─────────┘
                 │  → Aktion)  │                 │                           │
                 └──────┬──────┘        ┌────────┴────────┐                  ▼
                        ▼               ▼                 ▼            Sprachmodell
                    Aktoren        Rohdaten        Langzeitstatistik    (Client, kein
                                   10 Tage         dauerhaft            Modell im Server
                                   volle Auflösung 5-Min-/Stundenwerte  hinterlegt)
```

Drei Dinge, die aus dem Bild ablesbar sind und leicht übersehen werden:

- **Zwei getrennte Zigbee-Netze.** Zigbee2MQTT und die Hue Bridge sprechen
  beide Zigbee, aber es sind zwei Netze mit zwei Koordinatoren, die
  voneinander nichts wissen. Das ist gewachsen, nicht entschieden — die Hue
  Bridge war am Tag der HA-Erstinstallation da (26.09.2025), Zigbee2MQTT kam
  acht Wochen später (18.11.2025). Siehe Abschnitt 8.
- **MQTT ist der einzige Pfad mit einem echten Bus.** Hue, Matter und Shelly
  gehen über integrationseigene Wege direkt nach HA. Nur bei Zigbee liegt
  zwischen Gerät und HA ein Broker, an den sich ein zweiter Konsument hängen
  kann.
- **Der Recorder hängt an HA, nicht am Bus.** Was HA nicht sieht, wird nicht
  aufgezeichnet. Ein Ausfall von Home Assistant ist damit auch eine Lücke in
  den Zeitreihen — nicht nur ein Ausfall der Automationen.

---

## 4. Zustand

Alle Werte ausgelesen am 26.08.2026.

### Plattform

| Feld | Wert |
|---|---|
| Installationsart | Home Assistant OS 18.2, als VM (KVM) auf dem Proxmox-Host aus `01-testumgebung.md` |
| Core | 2026.8.1 · Supervisor 2026.07.5 · Docker 29.6.2 |
| Zustand | RUNNING, healthy, supported, NTP synchron |
| Datenträger der VM | 30,8 GB gesamt, 16,1 GB belegt |
| Struktur | 15 Bereiche auf 3 Etagen, keine unzugeordnet |

### Geräte nach Anbindung

| Anbindung | Geräte | Entitäten | lokal? |
|---|---|---|---|
| Zigbee über Zigbee2MQTT | 8 + Bridge | 86 | ja |
| Zigbee über Hue Bridge | 8 | 26 | ja |
| Matter (Thread und WLAN) | 10 | 64 | ja |
| Shelly über WLAN | 10 | 89 | ja |
| Sonos, TV, DLNA, Drucker, 3D-Drucker | 16 | 148 | ja / teils unklar |
| Küchengeräte, Saugroboter | 8 | 130 | **nein, Cloud** |
| Paketverfolgung, Abfuhrkalender, Wetter | 3 | 14 | **nein, Cloud** |
| Telefone (HA-App) | 2 | 37 | ja |
| Netzwerk-Infrastruktur (Router, Repeater) | 54 | 134 | ja |
| Add-on-Verwaltung | 17 | 17 | ja |
| **Geräteregister gesamt** | **201** | **927** | |

**Die Zahl 927 braucht eine Einordnung, sonst ist sie irreführend.** Die
Router-Integration legt für *jeden* Client im Netz einen Anwesenheits-Tracker
und einen Internetzugang-Schalter an: **107 der 927 Entitäten sind reine
Netzwerk-Clients**, keine Hausautomation. Sie erklären auch, warum 230
Entitäten auf `unavailable` oder `unknown` stehen — Geräte, die gerade nicht
im Netz sind.

### Entitäten je Domain

| Domain | Anzahl | Domain | Anzahl |
|---|---|---|---|
| `sensor` | 305 | `climate` | 6 |
| `switch` | 175 | `media_player` | 6 |
| `device_tracker` | 108 | `script` | 4 |
| `binary_sensor` | 90 | `time` | 4 |
| `update` | 47 | `person`, `notify`, `vacuum` | je 2 |
| `button` | 45 | `input_boolean`, `input_select` | je 1 |
| `number` | 28 | `conversation`, `zone`, `sun` | je 1 |
| `select` | 27 | `remote`, `weather`, `todo` | je 1 |
| `event` | 17 | `tts`, `calendar` | je 1 |
| `scene` | 16 | | |
| `light` | 11 | | |
| `automation` | 8 | | |
| `cover`, `image` | je 7 | | |

Von den 90 Binärsensoren sind 35 Störungsmelder (`problem`), 14
Verbindungszustände und 14 Laufzustände von Geräten. Nur 3 melden Präsenz
oder Belegung, 2 melden Kontakte.

### Add-ons

14 installiert, 13 laufen, 3 Updates offen.

| Add-on | Rolle |
|---|---|
| Mosquitto broker | MQTT-Broker |
| Zigbee2MQTT | Zigbee-Anbindung |
| Matter Server | Matter-Controller |
| OpenThread Border Router | Thread-Netz |
| Home-Assistant-Matter-Hub | exportiert HA-Entitäten **an** einen Matter-Controller |
| Home Assistant MCP Server | Modellanbindung, → Abschnitt 7 |
| Webhook-Proxy für den MCP-Server | Fernzugriff, **nicht konfiguriert**, → Abschnitt 8 |
| Whisper, Assist Microphone | Spracherkennung, **nicht verdrahtet**, → Abschnitt 8 |
| Paperless-ngx | → `02-paperless.md` |
| Advanced SSH, File editor, Samba | Wartung und Dateizugriff |

### Was in Betrieb ist und was nur installiert

Der Unterschied ist wichtig genug, um ihn getrennt zu benennen:

| | Zustand |
|---|---|
| Geräteanbindung, Recorder, MCP-Server | **läuft** |
| Automationen | **3 von 8 laufen** (→ Abschnitt 6) |
| Automatische Backups | **läuft**, aber nur an einen Ort (→ Abschnitt 8) |
| Sprachein- und -ausgabe | installiert, **nicht in Betrieb** (→ Abschnitt 8) |
| Fernzugriff auf den MCP-Server | installiert, **nicht konfiguriert** (→ Abschnitt 8) |
| Monitoring | **gibt es nicht** (→ Abschnitt 8) |

---

## 5. Zeitreihen

### Zwei Stufen, nicht eine

Home Assistant hält Verlaufsdaten in zwei getrennten Formen. Das ist für die
Auswertung der wichtigere Punkt als jede Aufbewahrungsdauer:

| | Rohdaten | Langzeitstatistik |
|---|---|---|
| Inhalt | jede einzelne Zustandsänderung | 5-Minuten- und Stundenwerte: Mittel, Min, Max bzw. Summe |
| Aufbewahrung | **gemessen 10 Tage** | dauerhaft, wird nicht gelöscht |
| Voraussetzung | keine | Sensor braucht eine `state_class` |
| taugt für | „warum ging um 06:12 das Licht an" | „wie warm war es im Juli" |

**Gemessen am 26.08.2026:** Eine Historienabfrage über 90 Tage liefert als
ältesten Rohwert den 16.08.2026, 04:12 — exakt 10 Tage vor dem Abfragezeitpunkt.
Eine Statistikabfrage über 180 Tage liefert dagegen Monatswerte bis zurück in
den Juni 2026. Die zweistufige Haltung funktioniert also wie vorgesehen.

Datenbank: **SQLite, 783 MiB.** Die Größe wächst nicht unbegrenzt, weil die
Rohdaten nach 10 Tagen wegfallen. Nur die Langzeitstatistik wächst dauerhaft
weiter, und zwar langsam.

### Was aufgezeichnet wird

**123 der 305 Sensoren haben eine `state_class` und fließen damit in die
Langzeitstatistik:**

| Größe | Sensoren | Art |
|---|---|---|
| Temperatur | 26 | Messwert |
| Energie (kWh) | 15 | fortlaufender Zähler |
| Luftfeuchte | 9 | Messwert |
| Leistung (W) | 9 | Messwert |
| Spannung | 8 | Messwert |
| Batteriestand | 8 | Messwert |
| Datenrate | 6 | Messwert |
| Beleuchtungsstärke | 3 | Messwert |
| Strom (A) | 2 | Messwert |
| Wasserdurchfluss | 2 | Messwert |
| ohne Klassifizierung | 29 | Mess- und Zählwerte |

Die übrigen 182 Sensoren sind Text-, Zeitstempel- und Aufzählungswerte
(Gerätezustände, Programmnamen, Firmware-Stände). Die liegen nur in den
Rohdaten und verschwinden nach 10 Tagen.

### Gemessene Änderungsraten

Gezählte Zustandsänderungen im Fenster 25.08.2026 12:18 bis 26.08.2026 12:18.
Das sind keine Poll-Intervalle von Home Assistant, sondern die Melderaten der
Geräte selbst.

| Quelle | Größe | Änderungen/24 h | ergibt |
|---|---|---|---|
| Zigbee-Präsenzsensor | Beleuchtungsstärke | 1.771 | rund alle 10–15 s |
| Zigbee-Steckdose | Leistung | 1.700 | 10-Sekunden-Takt |
| Zigbee-Steckdose | Leistung | 955 | 10-Sekunden-Takt, nur unter Last |
| Matter-Heizkörperthermostat | Temperatur | 536 | rund alle 3 min |
| Matter-Temperatursensor | Temperatur | 192 | rund alle 7 min |
| Zigbee-Präsenzsensor | Belegung (binär) | 120 | ereignisgetrieben |
| Matter-Präsenzsensor | Belegung (binär) | 63 | ereignisgetrieben |
| Matter-Präsenzsensor | Temperatur | 25 | rund stündlich |
| Matter-Präsenzsensor | Luftfeuchte | 25 | rund stündlich |
| Shelly-Aktor | Leistung | 17 | nur während einer Fahrt |
| Shelly-Aktor | Rollo-Zustand | 11 | ereignisgetrieben |
| Shelly-Aktor | Energie | 5 | bei Änderung |
| Zigbee-Kontaktsensor | Gerätetemperatur | 7 | rund alle 3 h |
| Zigbee-Ventil | Wasserdurchfluss | 3 | nur bei Durchfluss |
| Zigbee-Steckdose | Energie (kWh) | 1 | bei Änderung der 2. Nachkommastelle |

**Die Spreizung ist der eigentliche Befund: Faktor 1.700 zwischen dem
schnellsten und dem langsamsten Sensor.** Zwei Geräte allein erzeugen mehr
Datenpunkte als der gesamte Rest zusammen. Wer die Datenbankgröße drücken
will, schließt diese beiden aus dem Recorder aus — nicht dreißig
Temperatursensoren.

### Wofür ausgewertet wird

Heute: Energiedaten über das Energie-Dashboard von Home Assistant
(15 Zählersensoren), und Verlaufsansichten bei der Fehlersuche an
Automationen.

Geplant, und der eigentliche Grund für die zweistufige Haltung: Die
Langzeitstatistik ist die Datenquelle für „wie gestern"-Anfragen an ein
Sprachmodell (→ `00-fundament.md`, Abschnitt 8). Kein Standardwerkzeug liest
sie aus; das braucht einen eigenen MCP-Server über die Statistics-API. Solange
den niemand geschrieben hat, ist die Statistik zwar vollständig, aber für ein
Modell nicht erreichbar.

---

## 6. Automationen

**8 Automationen. 3 laufen, 5 sind abgeschaltet.** Dazu 4 Skripte
(Filmabend, Normalzustand, Gartenbewässerung starten und stoppen).

Verwendete Auslöser: `state` (4×, teils mit Haltedauer), `device` (4×),
`time` (3×), `sun` (2×). Verwendete Bedingungen: Wochentag, Sonnenstand,
Entitätszustand, Auslöser-Kennung, Template. Ausführungsmodi: 7× `single`,
1× `restart`.

### 6.1 Der Befund vor den Einzelfällen

Anlagedatum, letzte Auslösung und Zustand, alles am 26.08.2026 ausgelesen:

| Automation | angelegt | zuletzt ausgelöst | Zustand | löst aus auf |
|---|---|---|---|---|
| Beleuchtung nach Rollo-Zustand | 05.04.2026 | 26.08.2026 | **läuft** | Bewegung |
| Licht an einer Innentür | 05.04.2026 | 25.08.2026 | **läuft** | Kontakt |
| Küchenlicht nach Präsenz | 27.06.2026 | 26.08.2026 | **läuft** | Präsenz |
| Rollos öffnen | 08.02.2026 | 25.05.2026 | aus | Sonnenaufgang |
| Rollos schließen, Szene, Schalter | 08.02.2026 | 30.05.2026 | aus | Sonnenuntergang |
| Abendroutine | 12.03.2026 | 25.05.2026 | aus | feste Uhrzeit |
| Morgenroutine | 22.06.2026 | 25.06.2026 | aus | zwei feste Uhrzeiten |
| Präsenz im Arbeitsraum | 30.06.2026 | 01.07.2026 | aus | Präsenz |

**Das Muster ist deutlich genug, um es zu benennen:**

- **Alle drei laufenden Automationen lösen auf einen Sensor aus.** Keine
  einzige zeit- oder sonnenstandsgesteuerte Automation ist noch aktiv.
- **Die beiden aufwendigsten sind die kurzlebigsten.** Die Morgenroutine
  lief drei Tage, die Arbeitsraum-Automation zwei. Beide sind die
  durchdachtesten Konstruktionen im Bestand — und beide sind aus.
- **Was überlebt hat, sind die kleinen.** Zwei der drei laufenden stammen
  vom selben Tag im April und tun jeweils genau eine Sache.

**Warum die fünf abgeschaltet wurden, ist nicht dokumentiert.** Dieses
Dokument stellt dazu keine Vermutung an. Es hält fest, was belegbar ist:
Fünf von acht Automationen wurden gebaut und wieder abgeschaltet, und die
Trennlinie verläuft nicht zwischen gut und schlecht gebaut, sondern zwischen
sensorgetrieben und zeitgetrieben — und zwischen „tut eine Sache" und „greift
in den Tagesablauf ein".

Die folgenden fünf Fälle sind deshalb interessant: die drei, die laufen,
und die zwei, die trotz besserer Konstruktion nicht überlebt haben.

### 6.2 Beleuchtung nach Rollo-Zustand — 05.04.2026, läuft

| | |
|---|---|
| **Auslöser** | Bewegung erkannt |
| | Bewegung weg, gehalten für **30 Sekunden** |
| **Bedingung** | im Aktionszweig: Zustand des Rollos |
| **Aktion** | Rollo geschlossen → Licht auf **5 %** |
| | sonst → Licht auf **100 %** |

**Warum die Bedingung der Rollo-Zustand ist und nicht die Uhrzeit.** Das ist
die eigentliche Entscheidung in dieser Automation. Naheliegend wären zwei
andere Bedingungen gewesen, beide schlechter:

- **Uhrzeit** („nach 22 Uhr gedimmt") liegt an jedem Wochenende falsch und
  bei jeder Abweichung vom Alltag.
- **Sonnenstand** (`sun.sun below_horizon`) liegt im Winter ab 16:30 falsch —
  dann wäre nachmittags gedimmt.

Der Rollo-Zustand ist der bessere Näherungswert, weil er nicht die Tageszeit
abbildet, sondern eine Absicht: **ein geschlossenes Rollo hat jemand
geschlossen.** Das ist ein Zustand, den ein Mensch gesetzt hat, kein
Umweltwert — und damit näher an dem, was die Automation eigentlich wissen
will.

Dass sie als einzige aus dem April-Bestand ohne Änderung durchläuft, ist ein
Indiz dafür, dass die Wahl trägt.

### 6.3 Licht an einer Innentür — 05.04.2026, läuft

Anonymisiert dokumentiert: Kontaktsensor an einer Innentür, geschaltet wird
eine Steckdose. Raum und Etage bleiben ungenannt.

| | |
|---|---|
| **Auslöser** | Kontakt öffnet · Kontakt schließt |
| **Modus** | `restart` — ein neuer Auslöser bricht den laufenden Ablauf ab |
| **Zweig 1** | Kontakt schließt **und** der vorherige Zustand hielt **weniger als 1 Sekunde** → sofort aus |
| **Zweig 2** | Kontakt öffnet **und** es ist nach Sonnenuntergang → an, 5 Minuten warten, aus |
| **Zweig 3** | Kontakt schließt → 10 Minuten warten, aus |

**Warum drei Zweige und warum diese Reihenfolge.** `choose` nimmt den ersten
Zweig, dessen Bedingung zutrifft. Die Reihenfolge *ist* deshalb die Logik:

- **Zweig 1 ist sichtbar eine Korrektur.** Eine Bedingung, die prüft, ob der
  vorherige Zustand weniger als eine Sekunde gehalten hat, schreibt niemand
  im ersten Entwurf. Sie fängt einen Kontakt ab, der beim Schließen prellt
  oder doppelt meldet — ohne sie liefe für so ein Doppelsignal der
  10-Minuten-Ablauf aus Zweig 3 an, und das Licht bliebe zehn Minuten stehen,
  obwohl niemand da war. Was genau das ausgelöst hat, ist **nicht
  dokumentiert**; dass es eine Reaktion auf beobachtetes Verhalten war, steht
  in der Bedingung selbst.
- **Zweig 2 schaltet nur nach Sonnenuntergang.** Tagsüber passiert nichts —
  eine Automation, die tagsüber Licht einschaltet, ist Stromverbrauch ohne
  Wirkung.
- **`restart` statt `single` ist notwendig, nicht kosmetisch.** Bei `single`
  würde ein zweiter Auslöser während der 5- oder 10-Minuten-Wartezeit
  verworfen: Der Ablauf des ersten Auslösers liefe zu Ende und schaltete aus,
  obwohl inzwischen wieder jemand da ist. `restart` verwirft stattdessen den
  alten Ablauf und startet den neuen. **Wer mit Wartezeiten in Automationen
  arbeitet, muss den Ausführungsmodus mitentscheiden** — das ist der
  häufigste stille Fehler in HA-Automationen, weil `single` der Standard ist
  und ohne Wartezeiten auch richtig.

### 6.4 Küchenlicht nach Präsenz — 27.06.2026, läuft

| | |
|---|---|
| **Auslöser** | Präsenz erkannt → Steckdose an |
| | Präsenz weg, Haltedauer **0 Sekunden** → Steckdose aus |
| **Bedingung** | keine |

Die einfachste Automation im Bestand: ein Sensor, ein Aktor, keine Bedingung,
keine Verzögerung. Sie läuft seit zwei Monaten unverändert und hat heute
ausgelöst.

**Die Ausschaltverzögerung fehlt — und das ist der interessante Teil.** Ohne
Haltedauer ist sie anfällig für das Verhalten, das 6.5 abfängt: Meldet der
Sensor beim Stillstehen kurz „frei", geht das Licht aus und beim nächsten
Schritt wieder an. In einer Küche, in der man selten fünf Minuten
regungslos steht, fällt das womöglich nicht auf.

Ob das eine bewusste Vereinfachung oder eine ausstehende Korrektur ist, ist
**nicht dokumentiert**. Der Vergleich mit 6.5 legt eine unbequeme Lesart
nahe, die ehrlicherweise dastehen sollte: Die Automation ohne Verzögerung
läuft seit zwei Monaten, die mit Verzögerung wurde nach zwei Tagen
abgeschaltet.

### 6.5 Präsenz im Arbeitsraum — 30.06.2026, nach zwei Tagen abgeschaltet

| | |
|---|---|
| **Auslöser** | Präsenz gemeldet, gehalten für **1 Minute** |
| | Keine Präsenz, gehalten für **5 Minuten** |
| **Bedingung** | keine |
| **Aktion** | Belegt: Steckdose an, **Rollo auf 30 %** |
| | Frei: Steckdose aus, **Rollo ganz auf** |

**Warum die Haltedauern unterschiedlich sind.** Das ist die durchdachteste
Stelle im ganzen Automationsbestand. Symmetrische Zeiten wären falsch:

- **1 Minute beim Einschalten** filtert das Durchqueren des Raums heraus.
  Ohne die Haltedauer schaltet jeder Weg zum Fenster den Arbeitsplatz ein.
- **5 Minuten beim Ausschalten** fangen die Schwäche des Sensors ab. Ein
  Präsenzradar erkennt Bewegung zuverlässig, ruhiges Sitzen dagegen nicht
  immer. Ohne die Verzögerung geht am Schreibtisch das Licht aus, während
  jemand daran sitzt — der klassische Fehler bei präsenzgesteuerter
  Beleuchtung.

**Und sie lief trotzdem nur zwei Tage.** Angelegt am 30.06.2026, zuletzt
ausgelöst am 01.07.2026. Der Grund ist **nicht dokumentiert**. Belegbar ist
nur ein Unterschied zur Küchen-Automation (6.4), die seither durchläuft:
Diese hier schaltet nicht nur eine Steckdose, sondern **fährt zusätzlich ein
Rollo auf 30 %**. Sie verändert also den Raum sichtbar, sobald jemand ihn
betritt, und stellt ihn beim Verlassen wieder um.

**Die Lehre, soweit sie sich belegen lässt:** Die Qualität der Bedingungen
hat über das Überleben dieser Automation nicht entschieden. Der Eingriffsgrad
könnte es getan haben. Eine Automation, die eine Steckdose schaltet, verzeiht
Fehler; eine, die ein Rollo bewegt, wird bei jedem Fehlauslöser bemerkt.

### 6.6 Morgenroutine — 22.06.2026, nach drei Tagen abgeschaltet

Die einzige mehrstufige Automation. Uhrzeiten bewusst nicht dokumentiert.

| | |
|---|---|
| **Auslöser** | zwei Zeitpunkte am frühen Morgen, **20 Minuten auseinander** |
| **Bedingung** | Montag bis Freitag |
| **Stufe 1** | Merker zurücksetzen · 7 Rollos öffnen · Kaffeemaschine an · Wettervorhersage abrufen · Nachricht aus Wetter- und Abfuhrdaten bauen · Push an zwei Telefone, **mit Abbruch-Schaltfläche** |
| **Stufe 2** | **nur wenn der Merker nicht gesetzt ist:** Lampe an · Lautstärke auf 10 % · Radiostream starten |

**Warum eine Automation mit zwei Auslösern und nicht zwei Automationen.**
Beide Stufen teilen sich die Wochentagsbedingung und den Merker. In zwei
Automationen stünde beides doppelt da — und würde beim nächsten Ändern
auseinanderlaufen.

**Warum ein Merker (`input_boolean`) und nicht eine Variable.**
HA-Automationen sind zustandslos: Was in einem Durchlauf passiert, ist im
nächsten vergessen. Wer über 20 Minuten hinweg etwas merken will, braucht
eine Entität, die den Zustand hält. Das ist der vorgesehene Weg, und er hat
einen Nebeneffekt, der ihn zusätzlich rechtfertigt: Der Merker ist sichtbar
und von Hand schaltbar.

**Warum der Abbruch überhaupt existiert — das ist der Kern.** Die Aufteilung
in eine leise Stufe (Rollos, Kaffee) und eine laute (Lampe, Radio) mit 20
Minuten dazwischen ist nicht Komfort, sondern die Konstruktion, die den
Abbruch erst möglich macht. Eine einstufige Automation hätte kein Zeitfenster,
in dem man widersprechen kann. Der Abbruch läuft über eine Schaltfläche in
der Push-Nachricht — also über denselben Kanal, der ohnehin schon aufgemacht
wurde, ohne zusätzliche App und ohne Sprachbefehl.

**Was sie besser machte als ihre Vorgängerin.** Am 08.02.2026 gab es eine
Automation, die morgens dieselben 7 Rollos öffnete; sie hat zuletzt am
25.05.2026 ausgelöst. Zwei sachliche Unterschiede:

- **Die alte löste auf Sonnenaufgang aus.** Der schwankt in Deutschland
  zwischen etwa 04:20 im Juni und 08:30 im Dezember. Als Kopplung an einen
  Tagesablauf ist das unbrauchbar — im Sommer stehen die Rollos vor vier Uhr
  offen, im Winter erst, wenn man aus dem Haus ist.
- **Die alte sprach die Rollos über `device_id` an**, die neue über
  `entity_id` (→ Abschnitt 8).

**Und auch sie lief nur drei Tage.** Angelegt am 22.06.2026, zuletzt
ausgelöst am 25.06.2026, seither aus. Der Grund ist **nicht dokumentiert**.

Das ist die ehrlichste Aussage, die dieses Dokument über den Stand der
Hausautomation machen kann: **Die Geräte- und Datenebene läuft stabil seit
elf Monaten. Die Automationsebene ist noch nicht angekommen** — was
zuverlässig läuft, sind drei kleine Sensor-Aktor-Kopplungen, und jeder
Versuch, daraus einen Tagesablauf zu bauen, wurde nach wenigen Tagen
zurückgenommen.

---

## 7. MCP-Anbindung

### Aufbau

Der MCP-Server läuft als Add-on **auf demselben Host wie Home Assistant** und
spricht dessen API von innen an. Er verlässt das lokale Netz nicht: Nabu Casa
ist nicht eingeloggt, ein installierter Webhook-Proxy für Fernzugriff ist
**nicht konfiguriert** (→ Abschnitt 8).

**Im Server ist kein Modell hinterlegt.** Er stellt Werkzeuge bereit; welches
Modell sie benutzt, entscheidet der verbindende Client. Diese Erhebung lief
über Claude; für ein lokales Modell auf eigener GPU (→ `00-fundament.md`,
Abschnitt 3) müsste an der Hausseite nichts geändert werden. Genau das war
der Grund für MCP (→ Abschnitt 2).

### Was exponiert ist

**78 Werkzeuge, keines abgeschaltet.**

| Gruppe | Anzahl | was sie tun |
|---|---|---|
| Lesend | 32 | Zustände, Historie, Geräte, Bereiche, Integrationen, Protokolle, Templates auswerten, suchen |
| Steuernd | 5 | Dienste aufrufen, Geräte schalten, Ereignisse auslösen, Aufgabenlisten pflegen |
| Konfigurationsändernd | 30 | Automationen, Skripte, Szenen, Helfer, Dashboards, Bereiche, Entitäten anlegen, ändern, löschen |
| Systemeingriff | 11 | Neustart, Konfiguration neu laden, Add-ons verwalten, Backups, Updates |

### Was das Modell darf — und was das heißt

**Derzeit: alles.** Der Nur-Lesen-Modus ist aus, kein Werkzeug ist gesperrt,
es sind keine Werkzeug-Richtlinien aktiv. Ein verbundenes Modell kann
Automationen ändern und löschen, Geräte aus dem Register entfernen, Add-ons
starten und stoppen und Home Assistant neu starten.

**Das ist eine Entscheidung, kein Versehen** — aber sie steht und fällt mit
dem, was dagegen steht:

1. **Rückrollbarkeit statt Einschränkung.** Vor jeder Änderung an
   Automationen, Skripten, Szenen, Helfern und Dashboards legt der Server
   automatisch einen Stand an, 100 Versionen je Entität. Eine kaputt
   geschriebene Automation ist damit ein Rückgängig, kein Schaden.
   **Grenze:** Das deckt Konfigurationsobjekte ab. Ein gelöschtes Gerät, ein
   gestopptes Add-on und ein Neustart fallen nicht darunter.
2. **Der Server verlässt das LAN nicht.** Wer ihn erreichen will, muss im
   Netz sein.
3. **Es gibt keine dritte Ebene.** Weder eine Bestätigungspflicht für
   schreibende Werkzeuge noch eine Einschränkung auf eine Werkzeugliste.

**Bewertung, damit sie dasteht:** Für eine Betriebsphase, in der ein Modell
beim Aufbau hilft, ist das vertretbar — Werkzeuge, die man abschaltet, kann
man nicht benutzen. Für einen Dauerbetrieb, in dem ein Sprachassistent im Haus
lauscht, ist es das nicht. Der Weg dorthin steht in Abschnitt 8.

### Nicht zu verwechseln: der Assist-Weg

Unabhängig vom MCP-Server exponiert Home Assistant **104 Entitäten** an seinen
eigenen Sprachagenten (Assist). Das ist die Liste, die ein Sprachbefehl
erreichen kann — Licht, Rollos, Szenen, Schalter, Thermostate,
Medienwiedergabe. Sie ist deutlich enger als das, was der MCP-Server kann, und
sie ist die richtige Ebene für den späteren Dauerbetrieb: **das Modell
entscheidet was, der Code entscheidet wie** (→ `00-fundament.md`,
Abschnitt 1).

Der Sprachpfad selbst ist allerdings noch nicht in Betrieb (→ Abschnitt 8).

---

## 8. Offene Punkte

Nichts hiervon läuft. Wo ein Ansatz steht, ist er als geplant gekennzeichnet.

### Die Automationsebene trägt noch nicht

Fünf der acht Automationen sind abgeschaltet (→ Abschnitt 6.1). Was
zuverlässig läuft, sind drei kleine Sensor-Aktor-Kopplungen. Jeder Versuch,
daraus einen Tagesablauf zu bauen — Morgenroutine, Abendroutine,
Arbeitsraum — wurde nach wenigen Tagen zurückgenommen.

**Warum, ist nicht dokumentiert, und das ist die eigentliche Lücke.** Ohne
den Grund lässt sich der nächste Versuch nicht besser machen als der letzte.

**Geplant:** Für jede abgeschaltete Automation in einem Satz festhalten, was
gestört hat, bevor sie gelöscht oder neu gebaut wird. Das ist keine
Fleißarbeit, sondern die Voraussetzung dafür, dass aus acht Einzelversuchen
eine Regelbasis wird — und später die Spezifikation dafür, was ein
Sprachassistent im Haus überhaupt tun soll (→ `00-fundament.md`,
Abschnitt 7).

### Backups: eine Kopie ist keine Kopie

Automatische Vollbackups **laufen**, täglich, verschlüsselt, mit Datenbank.
Der letzte erfolgreiche Lauf war am 26.08.2026 um 03:37 Uhr.

**Das Problem ist das Ziel: `hassio.local` — und sonst nichts.** Alle Stände
liegen auf derselben Maschine wie die Installation. Fällt der Host aus, sind
Installation und Backup zusammen weg. Das ist derselbe Denkfehler wie „Parity
ist kein Backup" (→ `00-fundament.md`, Abschnitt 6), nur eine Ebene höher.

**Geplant:** Zweites Backup-Ziel auf dem Unraid-Share `backups/homeassistant/`,
das dort ohnehin vorgesehen ist. Damit wäre die Kette dieselbe wie für alle
anderen Dienste, statt einer Sonderlösung. Bis die Unraid-Kiste steht, ist das
nicht auflösbar — vorher wäre jedes Ziel provisorisch.

### Monitoring: gibt es nicht

Keine Benachrichtigung, wenn ein Sensor ausfällt, der Broker wegbleibt oder
eine Automation nicht mehr auslöst. Aktuell stehen **230 der 927 Entitäten**
auf `unavailable` oder `unknown`. Der größte Teil davon sind Netzwerk-Clients,
die schlicht nicht zu Hause sind — aber genau deshalb fällt ein echter Ausfall
darin nicht auf.

**Geplant:** Der ehrliche erste Schritt ist keine Monitoring-Lösung, sondern
eine Liste. Welche 20 bis 30 Entitäten sind kritisch — Broker, Koordinator,
Präsenzsensoren, Aktoren an aktiven Automationen? Erst wenn die steht, ist
eine Überwachung auf `unavailable` mit Push-Benachrichtigung sinnvoll.
Andernfalls überwacht man 927 Entitäten und schaltet die Benachrichtigungen
nach drei Tagen ab.

### Sprachpfad: installiert, nicht verdrahtet

Whisper und der Mikrofon-Satellit laufen als Add-ons. Die Assist-Pipeline ist
aber unangetastet: **kein Spracherkennungs-Dienst, kein Sprachausgabe-Dienst
und kein Aktivierungswort eingetragen, Pipeline-Sprache `en`**, obwohl Home
Assistant auf Deutsch läuft.

Es gibt also derzeit **keine funktionierende Sprachein- oder -ausgabe.** Die
Bausteine liegen bereit, sie sind nur nicht verbunden.

**Geplant:** Pipeline auf Deutsch stellen, Whisper als Spracherkennung
eintragen, eine lokale Sprachausgabe ergänzen (derzeit ist nur die von Google
konfiguriert — die widerspricht der Lokal-Entscheidung aus Abschnitt 2).
Reihenfolge nach `00-fundament.md`, Abschnitt 7: erst wenn HA steht.

### MCP-Rechte einschränken

Solange ein Modell beim Aufbau hilft, sind alle 78 Werkzeuge offen richtig.
Für den Dauerbetrieb nicht.

**Geplant, in dieser Reihenfolge:**
1. Systemeingriffe abschalten — Neustart, Add-on-Verwaltung und Update-Steuerung
   braucht ein Assistent im Alltag nicht.
2. Löschende Werkzeuge abschalten.
3. Danach prüfen, ob steuernde Werkzeuge und der Assist-Weg (Abschnitt 7)
   dasselbe abdecken. Wenn ja, kann der MCP-Server für den Alltagsbetrieb auf
   lesend gestellt werden.

### Fernzugriff auf den MCP-Server

Ein Webhook-Proxy für Fernzugriff ist installiert und gestartet, aber **weder
mit einer externen Adresse noch mit einer Anmeldung konfiguriert.** Der Server
ist damit praktisch nur lokal erreichbar — das entspricht der Entscheidung aus
Abschnitt 2, ist aber Zufall und nicht Absicht.

**Geplant:** Entweder das Add-on entfernen, weil es nicht gebraucht wird, oder
— falls Fernzugriff gewünscht ist — **OAuth aktivieren, bevor eine externe
Adresse eingetragen wird**, nicht danach. Ein erreichbarer Endpunkt ohne
Anmeldung ist genau der Fehler, den `00-fundament.md`, Abschnitt 7 für
Paperless mit Tailscale statt Portfreigabe vermeidet.

### Automationen hängen an Geräte-IDs statt an Entitäten

Sieben der acht Automationen sprechen Geräte über ihre interne Geräte-ID an,
nicht über die Entität. Der Unterschied ist beim Schreiben unsichtbar und beim
Gerätetausch entscheidend: **Ein ersetztes Gerät bekommt eine neue ID. Die
Automation zeigt danach ins Leere — ohne Fehlermeldung.** Sie läuft, sie tut
nur nichts.

Die Morgenroutine (6.6) macht es bereits richtig und ist damit die Vorlage.

**Geplant:** Bei der nächsten inhaltlichen Änderung je Automation umstellen,
nicht als eigene Aktion. Ein Umbau aller sieben auf einmal ohne Anlass tauscht
ein latentes Risiko gegen ein akutes.

### Zwei Zigbee-Netze

Zigbee2MQTT und die Hue Bridge betreiben zwei getrennte Netze mit je 8
Geräten und je einem eigenen Koordinator. **Das ist gewachsen, nicht
entschieden** — die Hue Bridge stand am Tag der HA-Erstinstallation, Z2M kam
acht Wochen später.

Zwei Netze auf denselben Funkkanälen sind nicht kostenlos: Sie konkurrieren um
Sendezeit, und es gibt zwei Stellen, an denen ein Zigbee-Problem auftreten
kann. Andererseits läuft die Bridge, und ein Umzug der Leuchten nach Z2M
bedeutet, jedes Gerät neu anzulernen und die Hue-Szenen zu verlieren.

**Geplant:** Vor dem nächsten Zigbee-Zukauf entscheiden, nicht danach. Bis
dahin gilt die faktische Regel: Neues geht nach Z2M.

### Recorder-Konfiguration

Die Aufbewahrung ist mit 10 Tagen gemessen. Ob ein `recorder:`-Block in
`configuration.yaml` existiert — mit `purge_keep_days` und mit Ausschlüssen —
ist **unklar** und über die API nicht feststellbar.

**Geplant:** Nachsehen und hier eintragen. Dann wird aus dem Zustand eine
Entscheidung, und die Ausschlussliste bekommt einen ersten Kandidaten: die
beiden Geräte, die im 10-Sekunden-Takt melden (→ Abschnitt 5).

### Kleinere offene Punkte

- **Zigbee-Router-Anzahl unklar.** Vier der acht Z2M-Geräte melden keinen
  Batteriestand und sind damit netzbetrieben, können also routen. Wie viele
  tatsächlich als Router im Netz stehen, veröffentlicht Zigbee2MQTT nicht
  über die Home-Assistant-Schnittstelle.
- **Bambu-Lab-Anbindung unklar** — lokal oder über die Cloud, aus Home
  Assistant nicht unterscheidbar. Relevant für Abschnitt 2.
- **Home-Assistant-Matter-Hub läuft**, exportiert also HA-Entitäten an einen
  Matter-Controller. An welchen, ist **unklar**. Falls das ein Sprachassistent
  eines Herstellers ist, gehört das in Abschnitt 2 zur Lokal-Entscheidung.
- **Drei Add-ons haben offene Updates.**
- **Eine deaktivierte Automation trägt noch ihren Testnamen.**

---

## Änderungslog

| Datum | Änderung |
|---|---|
| 26.08.2026 | Erstfassung. Zahlen aus der laufenden Installation erhoben, nicht geschätzt. Entscheidungen zu Home Assistant, Protokollwahl, Lokalbetrieb, Zigbee2MQTT statt ZHA, Netzwerk-Koordinator und MCP dokumentiert. Recorder-Aufbewahrung bewusst als Zustand und nicht als Entscheidung geführt, solange die Konfiguration nicht eingesehen ist. |
| 26.08.2026 | Automationen mit Anlagedatum, letzter Auslösung und Zustand aufgenommen. Befund: 5 von 8 abgeschaltet, die aufwendigsten am kürzesten gelaufen. Grund nicht dokumentiert, als offener Punkt geführt. |
| 26.08.2026 | Korrektur gegenüber der ursprünglichen Annahme: Backups laufen. Die Lücke ist das einzige Ablageziel, nicht das fehlende Backup. Monitoring fehlt tatsächlich. |
