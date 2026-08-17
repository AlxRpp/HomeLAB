# Homelab-Fundament

**Stand:** 16.08.2026 · **Status:** Planung, noch keine Hardware gekauft

---

## 1. Ziel

Ein Fundament für Homelab und Datenmanagement, das über Jahre nach Bedarf wächst.
Kernanforderungen:

- Ausfallsichere Datenhaltung, Platten einzeln nachrüstbar
- Selbst entworfene, netzwerkweit zugängliche Ordnerstruktur
- Vollständige Wiederherstellbarkeit auf neuer Hardware ("Bare-Metal-Restore")
- Erweiterbar per GPU, Platten und Software, ohne Neuaufbau
- Langfristig: lokaler Sprachassistent ("Jarvis") als steuerndes Gehirn

### Zielbild Jarvis

Vier Ebenen, von oben nach unten:

1. **Sprache** — Satelliten, STT, TTS (Wyoming-Protokoll)
2. **Jarvis Core** — lokales Modell, Tool-Routing
3. **MCP-Plugins** — Home Assistant, Paperless, qmd, später Kalender/Kamera
4. **Fundament** — Unraid, Shares, Parity, Backups

Leitprinzip: **Das Modell entscheidet *was*, der Code entscheidet *wie*.**
Deterministische Abläufe (Ausgabe-Routing, Kamera-auf-TV) sind Skripte,
keine LLM-Aufgaben. Sonst wird es unzuverlässig.

Beispiel-Anfragen als Zielmarke:
- "Mach die Gartenbewässerung für 12 Minuten an"
- "Mir ist kalt hier" → erkennt Raum über Präsenzsensor
- "Wie teuer ist meine Kfz-Steuer im Jahr?" → aus indexierten Dokumenten
- "Trag das in meinen Kalender ein" → Multi-Step über zweites Plugin
- "Wer steht vor der Tür?" → Ausgabe je nach Aufenthaltsort

---

## 2. Entscheidungen

### Unraid statt Proxmox als Basis

Frühere Empfehlung war Proxmox — begründet mit besserem VM-Backup/Restore.
Das stimmt weiterhin, war aber die falsche Frage. Entscheidend ist hier
Datenverwaltung, und da ist Unraid überlegen:

- **Platten einzeln nachrüstbar, gemischte Größen.** Kernfunktion von Unraid.
  ZFS-RAIDZ lässt sich nicht ohne Weiteres um eine Platte erweitern.
- **Shares als natives Konzept.** SMB-Freigabe per Klick, kein Samba-Config,
  kein LXC, kein Bind-Mount.
- **Docker-Container teilen sich eine GPU.** Ollama + Immich + Frigate +
  Jellyfin auf derselben Karte. Bei Proxmox-Passthrough bekommt genau eine
  VM die Karte. Für das Jarvis-Zielbild entscheidend.
- **Ausfallverhalten.** Fallen zwei Platten aus, sind nur deren Daten weg —
  jede Platte hat ein eigenes lesbares Dateisystem. Bei RAIDZ wäre alles weg.
- **Spindown.** Jede Datei liegt komplett auf einer Platte, also läuft beim
  Zugriff nur diese an. Spart real ~50 €/Jahr gegenüber ZFS.

### Eine Maschine statt zwei

Alles auf eine Kiste, NUC entfällt. Zwei Systeme hießen zwei Backup-Strategien,
zwei Update-Zyklen, zwei Systeme zum Verstehen — Gegenteil des Ziels.

Kein Stolperstein beim Zigbee-Koordinator: **SLZB-MR1 ist netzwerkbasiert**,
kein USB-Dongle-Passthrough nötig.

Preis dafür: Bei Unraid-Updates ist das Smart Home ein paar Minuten weg.
Etwa monatlich, akzeptabel.

**Bedingung: USV ist Pflicht**, kein Optional. Auf dieser Kiste liegen Fotos,
Dokumente, Backups und Haussteuerung. Stromausfall während eines Parity-Writes
ist das reale Risiko. Unraid unterstützt USVs nativ über NUT.

### Intel LGA1700 statt AM5

Frühere Empfehlung war AM5 — galt für eine reine Compute-Kiste. Für eine
24/7-Maschine dreht sich das:

- **Idle:** LGA1700 15–25 W, AM5 35–45 W durch das I/O-Die.
  Differenz ≈ 25 W ≈ **60 €/Jahr für nichts**.
- **Quick Sync:** UHD 770 transcodiert Jellyfin und beschleunigt Immich,
  ohne die große GPU zu wecken. AMDs VAAPI-Stack ist deutlich zickiger.
- **Sockel-Langlebigkeit irrelevant:** Der Upgrade-Pfad läuft über die GPU,
  nicht über die CPU.

### Fritz!NAS verworfen

10–25 MB/s, keine Redundanz, keine Snapshots, zickiges SMB, und Docker-Container
können nicht darauf laufen. Kein tragfähiges Fundament.

---

## 3. Hardware

### Geplante Konfiguration

| Teil | Auswahl | Anmerkung |
|---|---|---|
| CPU | Intel Core i5-14500 | 6P + 8E, ~200–230 € |
| Board | B760 / H770, mATX oder ATX | **≥ 6 SATA + zweiter PCIe-Slot** |
| RAM | 64 GB DDR5 (2× 32) | nicht kleiner |
| Netzteil | 550–650 W, 80+ Gold | Effizienz bei *niedriger* Last zählt |
| Gehäuse | Fractal Define 7 / Node 804 | 8+ Platten, GPU in voller Länge |
| Cache | 2× 1 TB NVMe, ZFS-Mirror | für `appdata` |
| Boot | guter USB-Stick | Lizenz an GUID gebunden |
| USV | Eaton/APC, ~150 € | Pflicht |

**Summe ≈ 900–1100 €** ohne Platten und GPU.

Kritisch beim Board: Viele moderne Boards haben nur 4 SATA-Ports.
Der zweite PCIe-Slot muss neben der GPU nutzbar bleiben.

### Nicht kaufen

Gebrauchte Büro-Kisten (Optiplex SFF/MT, EliteDesk). Kein Platz für eine
Grafikkarte in voller Länge → Kiste müsste zweimal gekauft werden.

### Ausbaupfad

- **GPU:** erst wenn CPU-Inferenz spürbar hakt. Ziel: gebrauchte RTX 3090,
  24 GB VRAM für 14B–32B-Modelle. 8B reicht für deutsches Tool-Calling nicht.
- **Platten:** Start 2× 14 TB (1 Parity + 1 Data). **Parity muss ≥ größte
  Datenplatte sein** — dort nicht sparen. Danach einzeln nachrüsten.
  Recertified Enterprise (Exos, Ultrastar) als Preis-Leistungs-Optimum.

---

## 4. Strom

Tarif **26,8 ct/kWh** → Faustformel: **1 W Dauerlast = 2,35 €/Jahr**

| Ausbaustufe | Dauerlast | Kosten/Jahr |
|---|---|---|
| Phase 1: 2 HDDs, keine GPU | 35–45 W | 85–105 € |
| Phase 2: 4 HDDs, 3090 idle | 70–90 W | 165–210 € |
| Volllast LLM (nur sekundenweise) | 250–400 W | vernachlässigbar |

Zum Vergleich: NUC bisher ≈ 30 €/Jahr.

### Sparmaßnahmen nach Wirkung

1. **C-States im BIOS** — größter Einzelhebel, meistübersehen. Werkseinstellung
   idlet oft bei 45 W statt 20 W. Package-C-State auf C8+, ASPM an,
   danach `powertop --auto-tune`. ≈ 60 €/Jahr für 20 Minuten Arbeit.
2. **Platten-Spindown** — 4 schlafende Platten ≈ 3 W statt ≈ 24 W. ≈ 50 €/Jahr.
3. **GPU erst bei Bedarf** — 3090 idle zieht 15–25 W ≈ 35–60 €/Jahr fürs Nichtstun.

---

## 5. Ordnerstruktur

Jeder Top-Level-Ordner unter `/mnt/user/` ist ein Unraid-Share.

```
/mnt/user/
├── appdata/              ← NUR Cache-Pool (NVMe), NIE ins Array
│   ├── paperless/
│   ├── immich/
│   └── jellyfin/
├── backups/
│   ├── flash/            ← Unraid-Config vom USB-Stick
│   ├── homeassistant/    ← HA schreibt direkt hierher
│   ├── appdata/          ← Appdata-Backup-Plugin, nächtlich
│   └── timemachine/
├── media/
│   ├── fotos/            ← Immich
│   ├── filme/            ← Jellyfin
│   ├── serien/
│   └── musik/
├── documents/
│   ├── alex/
│   ├── karina/
│   ├── gemeinsam/
│   └── homelab/          ← diese Datei, später von qmd indexiert
└── isos/
```

**Harte Regel:** `appdata` gehört auf den NVMe-Cache, nicht ins Array.
Dort liegen die Datenbanken von Paperless und Immich. SQLite/PostgreSQL auf
einem Parity-Array ist langsam und im schlimmsten Fall korrupt.
Dasselbe gilt für den Share `system` (enthält `docker.img`).

**Umkehrschluss — Nutzdaten gehören nicht auf den Cache.** Bei Paperless
heißt das: Datenbank und Suchindex nach `appdata`, aber die eigentlichen
Dokumente nach `documents/paperless/media`. Nur dort sind sie
parity-geschützt und Teil des Offsite-Backups. Analog später bei Immich:
Datenbank auf Cache, Bilder nach `media/fotos`.

---

## 6. Backup und Wiederherstellung

### Die drei Bestandteile

1. **Flash-Backup** — ZIP von `/boot`. Enthält die *komplette* Systemkonfiguration:
   Array-Zuordnung, Shares, Benutzer, Docker-Templates, VM-XMLs, Netzwerk.
   Wenige MB. Automatisch über Unraid-Connect-Account (kostenlos) oder manuell
   über Main → Flash → Flash Backup.
   Kopien: `backups/flash/`, Mac, offsite.
2. **Die Platten** — Zuordnung läuft über Seriennummern, Reihenfolge egal.
3. **Appdata-Backup** — liegt *nicht* auf dem Stick. Plugin fährt Container
   nächtlich runter, packt `/mnt/user/appdata` ins Array.

### Restore-Ablauf auf neuer Hardware

Neuer Stick → Unraid drauf → `config`-Ordner reinkopieren → Platten anstecken →
Lizenz auf neue GUID übertragen (einmal jährlich selbst über Web-Interface) →
Array starten. Container kommen aus Templates + Appdata-Backup zurück.

**Testbar in einer VM**, ohne Hardware und ohne Risiko: frische Unraid-VM,
echtes Flash-Backup einspielen, prüfen ob alles wiederkommt.

### Schlüssel sind Teil des Backups

Aus dem Paperless-Umzug am 16.08.2026 gelernt (`02-paperless.md`, Abschnitt 7):
Dienste verschlüsseln Zugangsdaten mit einem eigenen Schlüssel. Fehlt der beim
Restore, sind die Daten zwar da, aber unlesbar — und man merkt es erst hinterher.

**Regel: Bei jedem neuen Dienst prüfen, ob es einen solchen Schlüssel gibt,
ihn selbst setzen statt generieren zu lassen, und ins Passwortmanager-Backup
aufnehmen.** Ein Datenbank-Backup allein reicht nicht.

Zweite Lehre daraus: **Ein Anwendungs-Export enthält den Inhalt, nicht die
Betriebsumgebung.** Umgebungsvariablen und Container-Konfiguration kommen
nicht mit — dafür ist das Flash-Backup zuständig.

### Wichtig

**Parity ist kein Backup.** Sie schützt gegen Plattenausfall — nicht gegen
versehentliches Löschen, Ransomware, Blitzschlag oder Brand.
`documents/` und `media/fotos/` brauchen eine dritte Kopie außer Haus
(Backblaze B2 oder Hetzner Storage Box, verschlüsselt via Duplicati).

---

## 7. Reihenfolge

1. **Unraid-Test-VM auf dem NUC** — Array bauen, Shares anlegen, Container
   installieren, Flash-Backup erzeugen, VM plattmachen, Flash-Backup
   zurückspielen. Der Restore-Ablauf wird hier geübt, wo nichts dranhängt.
   *Grenze:* HA lässt sich darin nicht testen — der NUC hat 16 GB RAM und
   davon läuft das produktive HAOS bereits. Verschachtelte VM geht sich
   nicht aus. Für Array, Shares, Docker und Flash-Restore reicht es.
   → **Details, Stick-GUID, VM-Parameter und Testplan: `01-testumgebung.md`**
   (Kosten: 0 € bis auf den USB-Stick. Unraid-Trial 30 Tage, zweimal um
   je 15 Tage verlängerbar.)
2. **Parallel, reine Lesearbeit:** Board-Modelle (≥ 6 SATA + zweiter PCIe),
   USV-Modell inkl. NUT-Kompatibilität, Unraid-Lizenzstufe.
3. Hardware kaufen — USV inklusive, nicht nachgelagert
4. Unraid aufsetzen, Cache-Pool, Shares, USV einbinden
5. **Restore-Test auf der echten Kiste**, solange sie noch leer ist
6. HA als VM, Backup einspielen
7. Paperless als Docker-Container. **Entscheidung 16.08.2026: produktiv bei
   null anfangen** statt zu migrieren — der HA-Bestand war eine Testumgebung.
   Der Export/Import-Weg ist trotzdem geübt und dokumentiert, weil er später
   für Umzüge und als Backup-Methode gebraucht wird.
   → Alles Weitere in **`02-paperless.md`**
8. Backup-Kette vollständig scharf schalten, Restore ein zweites Mal
   testen — diesmal mit echten Daten
9. Erst dann: qmd, Ollama, Wyoming-Stack

### Wichtigster Sequenz-Hinweis: Core NICHT zuerst bauen

**Home Assistant ist bereits ein MCP-Client** (Integration "Model Context
Protocol"). HA pollt den SSE-Endpunkt eines MCP-Servers, holt die Tool-Liste
und stellt sie den Conversation Agents zur Verfügung.

Die Plugin-Architektur existiert also schon. MCP-Server für qmd schreiben,
in HA eintragen — fertig, ohne eine Zeile Core-Code.

Andersrum (Core zuerst) hätte man nach drei Monaten ein halbfertiges Framework
und keinen sprechenden Assistenten. So hat man nach zwei Wochen einen
funktionierenden Jarvis und lernt beim Benutzen, wo HA an Grenzen stößt —
*diese* Grenzen sind dann die Spezifikation für den eigenen Core.
Der Core wird später ein eigener Conversation Agent *in* HA, kein Ersatz für HA.

**Schnelle lokale Intents in HA aktiv lassen.** "Licht aus" muss in 200 ms
passieren und darf nicht von einem 8B-Modell abhängen. LLM nur als Fallback,
wenn der Intent-Parser nichts erkennt.

### Paperless als PWA

- HTTPS mit gültigem Zertifikat nötig, sonst installiert iOS die PWA nicht.
  Subdomain bei All-Inkl, Caddy oder NPM mit **DNS-01-Challenge** —
  funktioniert ohne Portfreigabe nach außen.
- Zugriff von unterwegs über **Tailscale**, nicht per Portfreigabe.
  Paperless enthält Ausweise, Verträge, Kontoauszüge.
- Safari → Teilen → Zum Home-Bildschirm.
- **Erfassung per Handy über den iOS-Scanner**, nicht als Foto. Paperless
  macht keine Rand- und Perspektivkorrektur — ein Foto mit Tischplatte und
  Schatten drückt die OCR-Rate deutlich. Dateien-App → *Dokumente scannen*
  liefert ein entzerrtes, sauberes PDF. Später ggf. als Kurzbefehl direkt
  an die Paperless-API.
- Paperless braucht **Redis als eigenen Container** (Queue für die OCR).
- `document_exporter` ist nicht nur der Migrationsweg, sondern auch die
  eigentliche **Backup-Methode**: ein versionsunabhängiges Archiv aus
  Dateien plus `manifest.json` mit allen Tags und Korrespondenten. Ein
  Appdata-Backup sichert nur die Datenbankdatei.

---

## 8. Bekannte Schwierigkeiten

- **"Wie gestern"-Anfragen:** Kein Standard-Tool liest HA-Historie aus.
  Braucht einen eigenen MCP-Server über die Recorder-/Statistics-API.
  Eigenes Projekt, kein Nebeneffekt.
- **Ausgabe-Routing** (Kamera auf TV je nach Raum): deterministisches Skript
  mit Raum-Zuordnung. Dem Modell nur ein Tool `zeige_kamera(kamera, ziel)` geben.
- **Deutsches Tool-Calling:** 8B-Modelle reichen nicht für mehrstufige Ketten.
  Realistisch 14B–32B → 24 GB VRAM.

---

## 9. Offene Punkte

- [ ] Konkrete Board-Modelle mit ≥ 6 SATA + zweitem PCIe-Slot recherchieren
- [ ] Unraid-Lizenz: Starter ($49, 6 Geräte) vs. Lifetime ($249, unbegrenzt).
      Bei geplantem Ausbau auf 2 NVMe + 4+ HDDs wird Starter schnell knapp.
      Unraid hat regelmäßig Sales (Sommer zuletzt 20–30 %).
- [ ] Offsite-Backup-Anbieter wählen
- [ ] USV-Modell wählen, NUT-Kompatibilität prüfen
- [x] ~~Migrationsplan Paperless aus dem HA-Add-on~~ — erledigt: Export/Import
      am 16.08.2026 getestet, produktiv wird neu aufgesetzt (`02-paperless.md`)
- [ ] Bestehendes Skript `~/scripts/apple_battery_health.sh` — offener Fix,
      unabhängig von diesem Projekt

---

## Änderungslog

| Datum | Änderung |
|---|---|
| 16.08.2026 | Erstfassung. Unraid statt Proxmox, eine Maschine, Intel statt AM5. |
| 16.08.2026 | Reihenfolge überarbeitet: Restore-Test vorgezogen (Übung in der Test-VM, echter Test vor Datenumzug). USV-Auswahl parallelisiert. Grenze der Test-VM dokumentiert. |
| 16.08.2026 | Test-VM begonnen. Details nach `01-testumgebung.md` ausgelagert. |
| 16.08.2026 | Abschnitt 5 ergänzt: Nutzdaten gehören nicht auf den Cache (Paperless-`media` nach `documents`, analog Immich). Share `system` ebenfalls cache-only. Abschnitt 7 (Paperless) ergänzt: iOS-Scanner statt Foto, Redis als eigener Container, `document_exporter` als Backup-Methode. |
| 16.08.2026 | Paperless-Wissen nach `02-paperless.md` ausgelagert. Export/Import aus dem HA-Add-on erfolgreich getestet (15 Dokumente inkl. Tags). Entscheidung: produktiv bei null anfangen statt migrieren. |
| 16.08.2026 | Abschnitt 6 um "Schlüssel sind Teil des Backups" ergänzt — verallgemeinerte Lehre aus dem Paperless-Secret-Key-Problem. Gilt für jeden künftigen Dienst. |
