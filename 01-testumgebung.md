# Testumgebung: Unraid-VM auf Proxmox

**Angelegt:** 16.08.2026 · **Status:** läuft — Array, Shares, Docker und
Paperless stehen. Offen: Flash-Backup und Restore-Test.
Gehört zu Schritt 1 aus `00-fundament.md`, Abschnitt 7.

**Zugriff:** `http://192.168.178.156` (DHCP) · Unraid 7.3.2

---

## 1. USB-Stick (Boot-Medium Test)

| Feld | Wert |
|---|---|
| Modell | Intenso Alu Line |
| **Device GUID (wie Unraid sie zeigt)** | **`058F-6387-0015-040100005124`** |
| Seriennummer (aus `lsusb`) | `15040100005124` |
| USB Vendor:Product ID | `058f:6387` (Alcor Micro) |
| Am Proxmox-Host als | `/dev/sda` |
| by-id | `usb-Intenso_Alu_Line_15040100005124-0:0` |

Die GUID ist die Kennung, an die eine Unraid-Lizenz gebunden wäre.
Für die Testinstanz läuft der **Trial**: 30 Tage ab Erstboot,
zweimal um je 15 Tage verlängerbar (= max. 60 Tage).

**Erstboot / Trial-Start:** 16.08.2026 → läuft bis ca. **15.09.2026**
(zweimal um je 15 Tage verlängerbar)

**Unraid.net-Account:** kostenlos, wird beim Trial-Start verlangt. Daran hängt
später die Lizenzverwaltung inkl. GUID-Übertragung auf einen neuen Stick —
also eine Mailadresse nehmen, die langfristig existiert. Liefert außerdem
**Unraid Connect** mit automatischem Flash-Backup (s. `00-fundament.md`,
Abschnitt 6) — in dieser VM gleich mittestbar.

> Der Stick der späteren echten Maschine bekommt eine eigene GUID
> und einen eigenen Trial. Diese hier ist reine Testumgebung.

### Prüfbefehle (Proxmox-Shell)

```bash
lsusb                                              # Stick finden
lsusb -v -d 058f:6387 2>/dev/null | grep -i iSerial # GUID auslesen
ls -l /dev/disk/by-id/ | grep -i usb
```

---

## 2. VM-Konfiguration

**Host:** NUC, Proxmox (Knoten `pve`) · **VM-ID:** `103` · **Name:** `UnRaid-TEST`

| Reiter | Einstellung |
|---|---|
| Allgemein | **"Beim Booten starten" AUS** — s. u. |
| OS | **Do not use any media** — Unraid hat keine Installer-ISO |
| System | Machine `q35`, BIOS **OVMF (UEFI)**, EFI-Disk hinzufügen |
| System | **"Pre-Enroll keys" abwählen** — sonst blockt Secure Boot den Boot |
| System | SCSI Controller: VirtIO SCSI single |
| CPU | 2 Kerne, Type `host` |
| Memory | 4096 MB, **Ballooning aus** |
| Network | Bridge `vmbr0`, Model VirtIO, DHCP |
| Disks | 4× 20 GB, Bus SCSI, mit Seriennummern (s. u.) |

### Virtuelle Platten

4× 20 GB: 1 Parity, 2 Data, 1 Cache. Die Proxmox-GUI hat **kein Feld für
Seriennummern** → per Shell, VM aus. `scsi0` kommt aus dem Assistenten und
bekommt die Serial nachträglich (existierendes Volume namentlich angeben,
dann wird nur die Drive-Zeile neu geschrieben, keine neue Platte):

```bash
qm set 103 -scsi0 local-lvm:vm-103-disk-1,iothread=1,size=20G,serial=PARITY01
qm set 103 -scsi1 local-lvm:20,serial=DATA01
qm set 103 -scsi2 local-lvm:20,serial=DATA02
qm set 103 -scsi3 local-lvm:20,serial=CACHE01
```

Unraid ordnet Platten **ausschließlich über Seriennummern** zu.
Genau dieser Mechanismus wird beim Restore-Test geprüft.

> **In der Praxis kamen die Serials nicht durch.** Unraid zeigt die Platten
> als `0QEMU_QEMU_HARDDISK_drive-scsiN` — QEMU meldet die interne Drive-ID
> statt des gesetzten `serial=`. Kein Blocker: Diese Namen sind ebenso
> eindeutig und stabil, der Restore-Test funktioniert damit. Auf echter
> Hardware stellt sich die Frage nicht (echte Herstellerseriennummern).

**Zuordnung in der Test-VM:**

| Unraid | Proxmox | Rolle |
|---|---|---|
| `drive-scsi0` (sda) | scsi0 | Parity |
| `drive-scsi1` (sdb) | scsi1 | Disk 1 |
| `drive-scsi2` (sdc) | scsi2 | Disk 2 |
| `drive-scsi3` (sdd) | scsi3 | Cache |

**Zwei Datenplatten statt einer**, weil erst damit das Kernverhalten sichtbar
wird: Jede Datei liegt komplett auf *einer* Platte, verteilt nach der
Allocation-Method des Shares. Das ist der Grund für die Entscheidung
gegen ZFS-RAIDZ (s. `00-fundament.md`, Abschnitt 2).

### Stick durchreichen und booten

```bash
qm set 103 -usb0 host=058f:6387
qm set 103 -boot order=usb0
qm config 103 | grep -E 'scsi[0-9]|usb|boot'   # Kontrolle
qm start 103
```

Vendor/Device ID statt USB-Port: funktioniert auch nach Portwechsel.
Falls `usb0` in der Bootreihenfolge abgelehnt wird: beim Start `Esc` →
OVMF Boot Manager → Stick wählen. OVMF merkt sich das in der EFI-Disk.

Entscheidend ist, dass **`scsi0` aus der Bootreihenfolge raus** ist. Sonst
bootet die VM von einer leeren Platte, bleibt bei "no bootable device"
stehen — und man sucht den Fehler beim Stick.

### Platzbedarf im Thin-Pool

`local-lvm` ist LVM-thin. Prüfen mit `pvesm status` und `lvs` (Spalte
`Data%` der LV `data`) — **nicht** mit `vgs`, das zeigt nur den Platz
*außerhalb* des Pools.

Stand 16.08.2026: Pool 140,87 GB, 22,5 % belegt. Die 4 Testplatten sind
nominell 80 GB, real belegt der Test ~30 GB (der Parity-Build schreibt die
Parity-Platte einmal komplett voll, der Rest bleibt fast leer).

> **Überbuchung ist bei Thin normal, aber:** Läuft ein Thin-Pool tatsächlich
> auf 100 %, hängen sich *alle* VMs darin gleichzeitig auf und nehmen
> Dateisystemschäden mit — nicht nur die, die zu viel geschrieben hat.
> Ab ~80 % `Data%` eingreifen.

---

## 3. Bewusst nicht gesetzt

- **Kein WLAN** im USB Creator. Die VM hat keine WLAN-Hardware, und ein
  24/7-Server gehört generell ans Kabel — Paketverluste mitten in einem
  SMB-Transfer, und bei Router-Neustart ist die Kiste weg.
- **Autostart aus.** Der NUC hat 16 GB und darauf läuft das produktive HAOS.
  Bei einem Neustart des NUC würde die Test-VM sich sonst 4 GB nehmen —
  das Smart Home darf nicht von einer Spielwiese abhängen. Auf der echten
  Kiste ist Autostart später richtig, hier nicht.
- **DHCP statt statischer IP.** Auf der echten Kiste später als
  **DHCP-Reservierung in der Fritzbox**, nicht statisch im Server:
  Die Fritzbox weiß dann, dass die Adresse vergeben ist, und die
  Netzkonfiguration liegt an einer Stelle statt auf zehn Geräten.

---

## 4. Unraid-Einrichtung (so gemacht)

### Array

`MAIN → Slots` zählt **alle Array-Slots inklusive beider Parity-Zeilen**,
nicht nur die Datenplatten. Für Parity + Disk 1 + Disk 2 also **Slots = 4**
(Parity, Parity 2, Disk 1, Disk 2). Bei Slots = 3 fehlt die Zeile für Disk 2.

**Parity 2 bleibt `unassigned`.** Dual Parity überlebt zwei gleichzeitige
Ausfälle, kostet aber eine komplette Platte. Bei 2 Datenplatten absurd.
Sinnvoll wird sie ab etwa 6–8 Datenplatten, weil dann das Risiko steigt,
dass während des tagelangen Rebuilds eine zweite Platte stirbt.

Cache als eigener Pool über `ADD POOL`, Name `cache`, 1 Slot.

Nach `START` melden sich die Platten als *"Unmountable: no file system"* —
das ist kein Fehler. Unraid formatiert nie ungefragt: Häkchen bei **Format**
setzen, bestätigen.

**Parity-Sync in der VM: 36 Sekunden bei 596 MB/s.** Aussagelos — Thin-Volumes
auf SSD, komplett leer. Auf echter Hardware **10–20 Stunden für 14 TB**,
unabhängig vom Füllstand. Genau deswegen ist die USV Pflicht.

### Benutzer

`alex` und `karina` als reine SMB-Konten. Klein geschrieben, keine Umlaute
(Unraid legt daraus Linux-Benutzer an). **Nicht** das root-Passwort verwenden —
root ist Admin für Oberfläche und SSH, das ist eine andere Ebene.

### Shares

**Alle Share-Namen klein schreiben.** Docker-Templates haben `/mnt/user/appdata`
hart eingetragen; bei `AppData` legt Unraid beim ersten Container klammheimlich
einen zweiten Share an.

| Share | Primary | Secondary | Export | Security |
|---|---|---|---|---|
| `appdata` | cache | **none** | **No** | – |
| `backups` | array | – | Yes | Private |
| `media` | cache | array | Yes | Secure |
| `documents` | cache | array | Yes | Private |
| `isos` | array | – | Yes | Secure |
| `system` | cache | none | No | – (legt Unraid selbst an) |

- **`appdata` nicht exportieren.** Der Finder würde `.DS_Store` in Verzeichnisse
  schreiben, die laufende Container offen halten → im schlimmsten Fall korrupte
  SQLite-Datei.
- **`system`** (enthält `docker.img`) muss ebenfalls cache-only sein, sonst
  wird alles zäh.
- **Secure vs. Private:** Secure = Lesen für alle frei, Schreiben nur für
  freigeschaltete Benutzer. Private = auch Lesen gesperrt.
  → Für Schreibzugriff auf `media`/`isos` muss `alex` explizit auf
  Read/Write stehen, sonst kann man im Finder keine Ordner anlegen.
- **Split level** bei `media`: *"Split only the top level directory as required"*.
  Wirkt **nicht rückwirkend**, also gleich richtig setzen.
- Unterordner (`documents/alex`, `media/fotos` …) im Finder anlegen, nicht in Unraid.

`SETTINGS → SMB → Enable: Yes`, dann am Mac `Cmd+K` → `smb://192.168.178.156`.

> Bereits gemountete Shares sind im macOS-Verbindungsdialog **ausgegraut**.
> Nach Rechteänderungen: Volume auswerfen, Server trennen, neu verbinden —
> macOS cached die Rechte.

### Docker

`SETTINGS → Docker → Enable: Yes`. Default appdata location `/mnt/user/appdata/`
lassen — das ist der Pfad, den alle Templates vorausgefüllt bekommen.

### Redis

Paperless-ngx braucht Redis als **separaten Container** (Queue für die
OCR-Verarbeitung). Aus Community Apps das großgeschriebene **"Redis"**
(Kategorie Network) nehmen — das kleingeschriebene `redis` verlangt eine
manuell angelegte `redis.conf`.

Defaults reichen. Läuft auf `bridge`, Port `6379` veröffentlicht
→ erreichbar als `redis://192.168.178.156:6379`.

### Paperless-ngx

| Feld | Wert |
|---|---|
| Data | `/mnt/user/appdata/paperless-ngx/data` |
| Media | `/mnt/user/documents/paperless/media` |
| Consumption | `/mnt/user/documents/paperless/consume` |
| Export | `/mnt/user/documents/paperless/export` |
| PAPERLESS_REDIS | `redis://192.168.178.156:6379` |
| PAPERLESS_OCR_LANGUAGE | `deu` |
| PAPERLESS_OCR_LANGUAGES | `deu` |
| PAPERLESS_TIME_ZONE | `Europe/Berlin` |

**Die Pfad-Aufteilung ist eine bewusste Entscheidung** (s. `00-fundament.md`,
Abschnitt 5):

- **data** = Datenbank, Suchindex → `appdata`/Cache. Viele kleine
  Schreibzugriffe, muss schnell sein.
- **media** = die eigentlichen Dokumente (Original + OCR-Version + Thumbnails)
  → `documents`/Array. Das ist das Archiv: parity-geschützt und im Offsite-Backup.
- **consume** = Briefkasten. Alles hier wird eingelesen und **danach gelöscht**.
  Nie etwas hineinlegen, von dem es keine Kopie gibt.
- **export** = Ziel für `document_exporter`. Im Normalbetrieb leer.

**Ordner vorher im Finder anlegen.** Docker legt fehlende Bind-Mount-Pfade
zwar an, aber als `root` — dann kommt man per SMB nicht mehr zum Schreiben rein.
Notfalls repariert `TOOLS → New Permissions`.

Zu den zwei OCR-Variablen: **Singular** wählt die benutzte Sprache,
**Plural** installiert die Sprachpakete beim Containerstart (Image bringt nur
Englisch mit). Sie können sich nicht in die Quere kommen. Für gemischte
Dokumente später den *Singular* auf `deu+eng` — kostet aber Genauigkeit
bei rein deutschen.

### Container Path vs. Host Path

Bind-Mount. **Container Path** gibt die Anwendung vor (Paperless sucht hart
codiert `/usr/src/paperless/media`) — nie ändern. **Host Path** bestimmt, wo
das physisch auf Unraid liegt. Deshalb sind Container ersetzbar: Löscht man
den Container, sind die Daten noch da — sie lagen nie in ihm.

### Dokumente vom Handy

Paperless macht **keine** Rand- und Perspektivkorrektur. Ein Foto mit
Tischplatte und Schatten drückt die OCR-Rate spürbar.

→ **iOS-Scanner benutzen:** Dateien-App → `...` → *Dokumente scannen*.
Kantenerkennung, Entzerrung, Helligkeitskorrektur, sauberes PDF. Dann per
*Teilen* in die Paperless-PWA.

Optional in Paperless: `PAPERLESS_OCR_DESKEW` (begradigt schiefe Scans) und
`PAPERLESS_OCR_ROTATE_PAGES`. Hilft bei schiefen Scans, nicht bei Fotos.

Idee für später: Kurzbefehl "scannen → per API an Paperless" für einen Tipp.

---

## 5. Testplan

Zweck ist **nicht** "Unraid anschauen", sondern den Restore-Ablauf zu üben,
bevor Daten dranhängen. Schritt 5 ist der eigentliche Test.

- [x] Array bauen: Parity + Data, Cache-Pool
- [x] **Ordnerstruktur aus `00-fundament.md` Abschnitt 5 anlegen**,
      inkl. `appdata` als Cache-only
- [x] SMB-Zugriff vom Mac prüfen (`Cmd+K` → `smb://<ip>`)
- [x] Container installieren (Redis + Paperless-ngx)
- [x] Ende-zu-Ende-Test: PDF im Finder nach `consume` → erscheint in Paperless
- [x] PWA auf dem iPhone, Upload von unterwegs über VPN
- [ ] Benutzerrechte durchspielen: `documents/alex` vs. `karina` vs. `gemeinsam`
      — prüfen, ob `karina` wirklich nicht sieht, was sie nicht soll
- [x] `document_exporter` / `document_importer` einmal durchspielen —
      15 Dokumente inkl. Tags aus dem HA-Add-on übernommen.
      **Vollständig dokumentiert in `02-paperless.md`, Abschnitte 7 und 8.**
- [ ] Flash-Backup ziehen und wegsichern
- [ ] Appdata-Backup-Plugin einrichten und einmal zurückspielen
- [ ] **VM plattmachen → neu aufsetzen → `config` zurückspielen →
      Platten anhängen → Array starten.** Kommt alles wieder?

### Grenzen dieser Umgebung

Nicht testbar, weil hardwareabhängig oder RAM-limitiert:
HA als verschachtelte VM, GPU/Quick Sync, echte Performance,
Platten-Spindown, USV/NUT.

Geschwindigkeit sagt hier nichts aus: Die "Platten" sind Dateien auf der
NUC-SSD, der NUC hat 1 GbE. Weder HDD-Verhalten noch Parity-Bremse sichtbar.

---

## 6. Notizen aus dem Testlauf

Was gehakt hat — Checkliste für die echte Kiste:

- **Trial muss aktiv sein, bevor Platten zuweisbar sind.** Ohne Keyfile steht
  `Slots` auf `none` und lässt sich nicht ändern. Der START-Button ist grau.
- Unraid 7 lässt ein **leeres Array starten**. Sieht aus wie Erfolg, ist aber
  nichts konfiguriert — Platten stehen weiter unter "Unassigned Devices".
- **`Slots` zählt beide Parity-Zeilen mit.** Slots = 3 zeigt nur Disk 1.
- Die Slot-Zeilen rendern gelegentlich nicht. Hartes Neuladen, oder Slots
  hoch- und wieder runterstellen.
- **`serial=` aus Proxmox kommt bei QEMU-SCSI-Platten nicht durch** (s. o.).
- Erst **Users**, dann Shares — sonst gibt es niemanden, dem man Rechte geben kann.
- Nach Rechteänderung im Finder **trennen und neu verbinden**, sonst greifen
  die alten Rechte weiter.
- **`.DS_Store`** breitet sich über SMB überall aus. Global abschaltbar:
  ```bash
  defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true
  # und für Git:
  git config --global core.excludesfile ~/.gitignore_global
  echo ".DS_Store" >> ~/.gitignore_global
  ```
- **Scheduled parity check ist ab Werk aus.** Auf der echten Kiste auf monatlich
  stellen. Bei Verdachtsfällen den ersten Check bewusst **ohne** "Write
  corrections to parity" laufen lassen — sonst überschreibt man eine korrekte
  Parity mit Daten von einer defekten Platte.
- **Sprache der Oberfläche auf Englisch lassen.** Die deutsche Übersetzung ist
  unvollständig, und jede Doku, jeder Forenbeitrag und jede Fehlermeldung ist
  englisch. "Freigabe" vs. "Share" kostet beim Suchen unnötig Zeit.

### Die drei wichtigsten Erkenntnisse aus dem Paperless-Umzug

Ausführlich in `02-paperless.md` — hier die Kurzfassung, weil sie über
Paperless hinaus gelten:

1. **`PAPERLESS_SECRET_KEY` von Anfang an selbst setzen und sichern.**
   Mail-Passwörter sind damit verschlüsselt. Anderer Key auf der Zielinstanz
   → Mail-Konfiguration ist unrettbar. Verallgemeinert: **Bei jedem Dienst
   prüfen, ob es einen Schlüssel gibt, ohne den ein Restore unvollständig
   bleibt** — und den ins Passwortmanager-Backup aufnehmen.
2. **Ein Export enthält den Inhalt, nicht die Betriebsumgebung.**
   Alles, was als Umgebungsvariable am Container hängt (OCR-Sprache,
   Dateinamen-Format, Zeitzone, Redis-Adresse), kommt *nicht* mit.
   Deshalb müssen Container-Templates separat gesichert werden — genau das
   leistet das Flash-Backup.
3. **Ein kaputter Teilbereich kann eine ganze Oberfläche lahmlegen.**
   Die 500er der Mail-API haben in Paperless dazu geführt, dass sich gar
   keine Dokumente mehr öffnen ließen. Bei unerklärlichen Frontend-Fehlern
   zuerst die Browser-Konsole auf 500er prüfen, statt am Symptom zu suchen.

---

## Änderungslog

| Datum | Änderung |
|---|---|
| 16.08.2026 | Erstfassung. Stick-GUID dokumentiert, VM-Parameter festgelegt. |
| 16.08.2026 | VM läuft. Array, Shares, Users, Docker, Redis, Paperless dokumentiert. Ende-zu-Ende-Test erfolgreich (Finder → consume → Paperless, PWA über VPN). Stolpersteine gesammelt. |
