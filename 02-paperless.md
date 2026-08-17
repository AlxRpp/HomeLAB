# Paperless-ngx

**Stand:** 16.08.2026 · **Status:** in der Test-VM erfolgreich aufgesetzt,
Export/Import aus dem HA-Add-on durchgespielt. Produktiv wird bei null begonnen.

Gehört zu `00-fundament.md` (Abschnitt 5 Ordnerstruktur, Abschnitt 7 Reihenfolge)
und `01-testumgebung.md` (Testaufbau).

---

## 1. Was Paperless-ngx ist

Ein Dokumentenarchiv. Man wirft PDFs oder Scans hinein, Paperless erkennt per
OCR den Text, macht das Dokument volltextdurchsuchbar und legt es strukturiert
ab. Verwaltet werden Dokumente über **Korrespondenten** (wer hat es geschickt),
**Tags** (frei vergebbare Schlagworte), **Dokumenttypen** (Rechnung, Vertrag,
Bescheid) und **Speicherpfade**.

Der Kerngedanke: **Man sucht, man sortiert nicht.** Ordner sind nur die
Rückfallebene für den Tag, an dem Paperless nicht läuft.

### Warum es im Homelab liegt und nicht in der Cloud

Paperless enthält Ausweise, Verträge, Kontoauszüge, Gehaltsabrechnungen.
Das ist der Datenbestand mit dem höchsten Schutzbedarf im ganzen Haus.
Zugriff von unterwegs deshalb ausschließlich über **Tailscale**, nie über
eine Portfreigabe.

---

## 2. Architektur

Paperless besteht aus **zwei Containern**:

| Container | Aufgabe |
|---|---|
| `paperless-ngx` | Web-Oberfläche, Datenbank, OCR-Verarbeitung |
| `Redis` | Queue für die OCR-Aufträge |

Redis ist keine Option, sondern Voraussetzung — Paperless legt jeden Scan als
Job in die Queue und arbeitet ihn asynchron ab. Ohne Redis startet es nicht.

In Unraid: Aus Community Apps das großgeschriebene **"Redis"** (Kategorie
Network) nehmen. Das kleingeschriebene `redis` verlangt eine manuell angelegte
`redis.conf`.

Redis läuft auf `bridge` mit veröffentlichtem Port `6379`
→ erreichbar als `redis://<unraid-ip>:6379`.

### Optional: Tika + Gotenberg

Zwei weitere Container, die Office-Dokumente (docx, xlsx, odt) und E-Mails in
PDF umwandeln, damit Paperless sie verarbeiten kann. Nur nötig, wenn nicht
ausschließlich PDFs und Bilder eingelesen werden. Aktuell nicht im Einsatz.

---

## 3. Die vier Verzeichnisse

Das ist die wichtigste Designentscheidung, und sie folgt direkt aus der
Cache/Array-Regel in `00-fundament.md` Abschnitt 5.

| Zweck | Container-Pfad | Host-Pfad | Warum dort |
|---|---|---|---|
| **data** | `/usr/src/paperless/data` | `/mnt/user/appdata/paperless-ngx/data` | Datenbank + Suchindex. Viele kleine Schreibzugriffe → **Cache/NVMe** |
| **media** | `/usr/src/paperless/media` | `/mnt/user/documents/paperless/media` | Die eigentlichen Dokumente → **Array**, parity-geschützt, im Offsite-Backup |
| **consume** | `/usr/src/paperless/consume` | `/mnt/user/documents/paperless/consume` | Briefkasten. Per SMB vom Mac/Handy erreichbar |
| **export** | `/usr/src/paperless/export` | `/mnt/user/documents/paperless/export` | Ziel für `document_exporter`. Im Normalbetrieb leer |

### Die Regel dahinter

**Datenbank auf den Cache, Nutzdaten ins Array.**

Eine SQLite- oder PostgreSQL-Datei auf einem Parity-Array ist langsam
(jeder Schreibvorgang kostet vier Operationen: alten Block lesen, alte Parity
lesen, Daten schreiben, Parity schreiben) und im schlimmsten Fall korrupt.

Umgekehrt haben die Dokumente auf dem Cache nichts verloren — dort gibt es
keinen Parity-Schutz und sie wären nicht Teil des Array-Backups.

Dieselbe Aufteilung gilt später bei Immich: Datenbank auf Cache,
Bilder nach `media/fotos`.

### `consume` — Vorsicht

Alles, was in `consume` landet, wird eingelesen und **danach gelöscht**.
Das ist gewollt (das Dokument liegt dann in `media`), aber:
**Nie etwas hineinlegen, von dem es keine zweite Kopie gibt.**

### Ordner vorher anlegen

Docker legt fehlende Bind-Mount-Pfade selbst an — aber als `root`. Dann kommt
man per SMB nicht mehr zum Schreiben rein, und genau das braucht man bei
`consume`. Also die Ordner vorher im Finder anlegen.

Reparatur, falls doch passiert: `TOOLS → New Permissions` in Unraid.

---

## 4. Konfiguration

Alle Einstellungen sind **Umgebungsvariablen am Container**, nicht in der
Datenbank. Das ist wichtig für Migration und Backup: Sie kommen bei einem
Export **nicht** mit (siehe Abschnitt 6).

| Variable | Wert | Bedeutung |
|---|---|---|
| `PAPERLESS_REDIS` | `redis://<unraid-ip>:6379` | Queue |
| `PAPERLESS_OCR_LANGUAGE` | `deu` | Welche Sprache die OCR **benutzt** |
| `PAPERLESS_OCR_LANGUAGES` | `deu` | Welche Sprachpakete beim Start **installiert** werden |
| `PAPERLESS_TIME_ZONE` | `Europe/Berlin` | |
| `PAPERLESS_FILENAME_FORMAT` | s. u. | Ablagestruktur in `media` |
| `PAPERLESS_SECRET_KEY` | **selbst setzen und sichern** | s. Abschnitt 7 |

### Die zwei OCR-Variablen

Sie werden regelmäßig verwechselt, können sich aber nicht in die Quere kommen:

- **Singular** (`_LANGUAGE`) wählt die benutzte Sprache
- **Plural** (`_LANGUAGES`) installiert Sprachpakete beim Containerstart —
  das Image bringt nur Englisch mit

Für gemischte Dokumente später den *Singular* auf `deu+eng` setzen. Das kostet
allerdings Genauigkeit bei rein deutschen Dokumenten, also erst bei Bedarf.

### Nützliche Zusatzvariablen

- `PAPERLESS_OCR_DESKEW` — begradigt leicht schiefe Scans
- `PAPERLESS_OCR_ROTATE_PAGES` — dreht falsch herum eingescannte Seiten

Beides hilft bei Scannern, **nicht** bei Handyfotos (siehe Abschnitt 5).

---

## 5. Ablagestruktur (`PAPERLESS_FILENAME_FORMAT`)

Bestimmt, wie Paperless die Dateien im `media`-Ordner physisch ablegt.

**Gewählt:**

```
{{ correspondent }}/{{ created_year }}/{{ created }}-{{ title }}
```

Ergibt:

```
media/
├── Finanzamt/
│   ├── 2025/
│   │   └── 2025-03-14-Steuerbescheid.pdf
│   └── 2026/
├── Stadtwerke/
└── Allianz/
```

**Begründung:** Die Ordnerstruktur ist nicht der Arbeitsweg — im Normalbetrieb
sucht man per Volltext. Sie ist die **Rückfallebene für den Tag, an dem
Paperless nicht läuft.** Und in dem Fall denkt man fast nie "2024", sondern
"Finanzamt". Deshalb Korrespondent zuerst, Jahr als zweite Ebene (damit ein
Korrespondent mit 200 Dokumenten übersichtlich bleibt), Datum im Dateinamen
für die chronologische Sortierung innerhalb des Ordners.

Jahr zuerst wäre nur besser, wenn regelmäßig **jahrgangsweise** gearbeitet
wird — etwa "alles aus 2025 für den Steuerberater".

> **Nicht nachträglich ändern.** Eine Umstellung sortiert alle vorhandenen
> Dateien neu ein. Vor dem produktiven Start entscheiden.

---

## 6. Dokumente erfassen

### Vom Mac

PDF im Finder nach `documents/paperless/consume` ziehen. Nach ein paar
Sekunden verschwindet sie dort und taucht in Paperless auf.

### Vom Handy — nicht fotografieren

**Paperless macht keine Rand- und Perspektivkorrektur.** Ein Foto mit
Tischplatte, schiefer Kante und Schatten geht genau so in die OCR und drückt
die Erkennungsrate deutlich.

Stattdessen den **iOS-Scanner** benutzen:
`Dateien-App → ... → Dokumente scannen`
(identisch auch aus der Notizen-App heraus)

Der erkennt Blattkanten automatisch, entzerrt die Perspektive, korrigiert
Helligkeit und liefert ein sauberes PDF. Dann über *Teilen* in die
Paperless-PWA hochladen.

**Idee für später:** Kurzbefehl "scannen → direkt per API an Paperless",
damit es ein Tipp statt fünf wird.

### Per Mail

Paperless kann ein Postfach überwachen und Anhänge automatisch einlesen
(Mail-Konto + Mail-Regel unter `Einstellungen → Mail`). Praktisch für
Rechnungen, die ohnehin per Mail kommen.

### Als PWA

- HTTPS mit gültigem Zertifikat nötig, sonst installiert iOS die PWA nicht.
  Subdomain bei All-Inkl, Caddy oder NPM mit **DNS-01-Challenge** — funktioniert
  ohne Portfreigabe nach außen.
- Safari → Teilen → Zum Home-Bildschirm
- Zugriff von unterwegs über **Tailscale**

---

## 7. `PAPERLESS_SECRET_KEY` — die wichtigste Lehre

**Paperless verschlüsselt Mail-Konto-Passwörter mit dem `PAPERLESS_SECRET_KEY`.**

Wird eine Instanz mit einem anderen Key aufgesetzt und ein Export eingespielt,
liegen die Mail-Passwörter verschlüsselt in der Datenbank, sind aber nicht
entschlüsselbar. Die Oberfläche quittiert das mit:

```
500 Internal Server Error
/api/mail_accounts/
/api/mail_rules/
```

Genau das ist beim Testimport passiert.

**Konsequenz für den produktiven Aufbau:**

1. `PAPERLESS_SECRET_KEY` von Anfang an **selbst setzen**, nicht generieren lassen
2. Im Passwortmanager sichern
3. In `01-testumgebung.md` bzw. der Doku der echten Kiste vermerken, dass er
   existiert (nicht den Wert selbst)

Ohne den Key ist ein Restore der Mail-Konfiguration nicht möglich.
Die Dokumente selbst sind davon **nicht** betroffen.

**Reparatur, wenn es passiert ist:** Entweder den Key der Quellinstanz
übernehmen und Container neu starten, oder die Mail-Konten löschen und neu
anlegen.

Key der Quellinstanz auslesen (Beispiel HA-Add-on):

```bash
docker exec <container-name> env | grep SECRET_KEY
```

---

## 8. Export und Import

`document_exporter` ist **beides**: der Migrationsweg *und* die eigentliche
Backup-Methode.

Ein Appdata-Backup sichert nur die Datenbankdatei — die ist bei einem
Versionswechsel oder einer Korruption nur bedingt hilfreich. Ein Export ist ein
**lesbares, versionsunabhängiges Archiv** aus Dateien plus `manifest.json`.

### Was mitkommt

Alles, was in der Datenbank steht:

- Dokumente inkl. Titel, Datum, Notizen, ASN
- Korrespondenten, Tags, Dokumenttypen, Speicherpfade
- Gespeicherte Ansichten, Dashboard-Einstellungen
- Benutzer, Gruppen, Berechtigungen (Passwörter als Hash)
- E-Mail-Konten und Mail-Regeln
- Workflows

### Was nicht mitkommt

Alles, was als Umgebungsvariable am Container hängt — also der komplette
Abschnitt 4 dieser Datei: OCR-Sprache, Dateinamen-Format, Zeitzone,
Redis-Adresse, Secret Key.

> **Merksatz:** Der Export enthält den *Inhalt*, nicht die *Betriebsumgebung*.

### Sicherheitshinweis

**Mail-Passwörter stehen im Export im Klartext.** Der Export-Ordner ist
entsprechend zu behandeln: nicht offen in einem Share liegen lassen, nicht
ungeschützt in ein Cloud-Backup schieben.

Für produktive Exporte gibt es `--passphrase`, das die sensiblen Felder
verschlüsselt.

### Ablauf: Export

Läuft **innerhalb** des Containers (Django-Management-Befehl, braucht die
Datenbank). Der Zielordner muss **vorher existieren** und dem Benutzer
`paperless` gehören — `docker exec` läuft als `root`, der Exporter als
`paperless`.

```bash
# Zielordner anlegen und Rechte setzen
docker exec <container> mkdir -p /pfad/zum/export
docker exec <container> chown paperless:paperless /pfad/zum/export

# Export
docker exec <container> document_exporter /pfad/zum/export

# Kontrolle: manifest.json + Dokumentdateien müssen da sein
docker exec <container> ls -la /pfad/zum/export
```

Der Export ist **nicht destruktiv** — die Quellinstanz bleibt unverändert.

### Ablauf: Import

**Die Zielinstanz muss leer sein.** Sonst bricht der Importer ab.

1. In der Zieloberfläche: `Dokumente` → alle auswählen → Löschen
2. **`Papierkorb` öffnen und endgültig leeren** — wird gern vergessen,
   gelöschte Dokumente zählen dort noch mit
3. Export-Ordner an den Zielpfad kopieren
4. Container-Console öffnen:

```bash
document_importer /usr/src/paperless/export
```

### Nach dem Import: Pflichtschritte

Der Importer baut **weder Suchindex noch Thumbnails** neu. Ohne diese Schritte
listet die Oberfläche die Dokumente zwar, findet sie beim Öffnen aber nicht:

```bash
document_index reindex     # Suchindex
document_thumbnails        # Vorschaubilder
document_sanity_checker    # Kontrolle: DB-Einträge ohne Datei?
```

`document_sanity_checker` muss "No issues detected" melden.

### Symptom: 404 beim Öffnen eines Dokuments

Tab-Titel zeigt *"undefined – Paperless-ngx"*, die Liste funktioniert aber.

**Ursache war nicht der Index**, sondern die kaputten Mail-Konten aus
Abschnitt 7: Die Angular-App lädt beim Start globale Daten inklusive
Mail-Konfiguration. Kommt dort ein 500, bricht die Initialisierung teilweise
ab und Dokument-IDs werden nicht sauber durchgereicht.

**Lösung** — Mail-Konten entfernen, dann Container neu starten:

```bash
python3 manage.py shell -c "from paperless_mail.models import MailRule, MailAccount; MailRule.objects.all().delete(); MailAccount.objects.all().delete()"
```

> **Merke:** Ein kaputter Teilbereich kann in Paperless die ganze Oberfläche
> lahmlegen, weil das Frontend beim Start alles auf einmal lädt. Bei
> unerklärlichen Frontend-Fehlern zuerst die Browser-Konsole auf 500er prüfen.

Nach einem Container-Neustart braucht Paperless **30–60 Sekunden**, bis Django,
Worker und Webserver stehen. Vorher liefert der Port nichts — in Unraid auf
`(healthy)` in der Uptime-Spalte warten.

Safari-Cache: `Cmd+Option+R` für hartes Neuladen, `Cmd+Option+E` leert den
Cache komplett (Entwickler-Menü muss aktiviert sein). Die PWA auf dem iPhone
hat einen **eigenen** Cache — dort App löschen und neu zum Homescreen hinzufügen.

---

## 9. Backup-Strategie

Drei Schichten, die jeweils abdecken, was die andere offen lässt:

| Was | Wogegen | Wie |
|---|---|---|
| **ZFS-Mirror** (Cache) | NVMe-Defekt | 2× NVMe, `appdata` liegt gespiegelt |
| **Parity** (Array) | HDD-Defekt | schützt `media` und `documents` |
| **Appdata-Backup** | Korruption, Fehlbedienung | Plugin, nächtlich, fährt Container runter, Archiv ins Array |
| **`document_exporter`** | Versionswechsel, Datenbankschaden | versionsunabhängiges Archiv |
| **Offsite** (B2/Hetzner) | Brand, Blitz, Ransomware | verschlüsselt via Duplicati |

**Warum das Appdata-Backup die Container runterfährt:** Eine laufende
Datenbank im laufenden Betrieb wegzukopieren ergibt oft eine Datei, die sich
nicht wiederherstellen lässt. Deshalb die paar Minuten Downtime nachts.

**Parity ist kein Backup.** Sie schützt gegen Plattenausfall, nicht gegen
versehentliches Löschen, Ransomware, Blitzschlag oder Brand.

---

## 10. Historie und Entscheidungen

### Bisher: Add-on in Home Assistant

- Add-on `ca5234a0_paperless-ngx` (BenoitAnastay), v4.0.0
- HAOS läuft als VM auf dem NUC unter Proxmox
- Dateinamen-Format war `{{ created_year }}/{{ correspondent }}/{{ title }}`
- Nur eine Handvoll Testdokumente

**Warum weg vom Add-on:** Paperless gehört nicht in die Smart-Home-Instanz.
Bei jedem HA-Update wäre das Dokumentenarchiv mit betroffen, die Daten liegen
auf der falschen Maschine ohne Parity, und die Ressourcen teilt es sich mit
der Haussteuerung. Auf Unraid ist es ein eigener Container mit eigenem
Lebenszyklus.

### Testlauf 16.08.2026

Export aus dem HA-Add-on, Import in den Unraid-Container. **15 Dokumente
inklusive Tags und Korrespondenten korrekt übernommen.** Mail-Konten kamen
mit, waren aber wegen des abweichenden Secret Keys nicht lesbar (Abschnitt 7).

### Entscheidung: produktiv bei null anfangen

Der bisherige Bestand war eine Testumgebung. Statt Altlasten zu migrieren
wird sauber neu aufgesetzt — mit von Anfang an durchdachten Korrespondenten,
Tags und Dokumenttypen.

Der Export/Import-Weg ist trotzdem geübt und dokumentiert, weil er später
für Umzüge und als Backup-Methode gebraucht wird.

---

## 11. Offene Punkte

- [ ] Tag- und Korrespondenten-Schema für den produktiven Aufbau festlegen
- [ ] `PAPERLESS_SECRET_KEY` erzeugen und im Passwortmanager sichern
- [ ] HTTPS via DNS-01-Challenge einrichten (Voraussetzung für die PWA)
- [ ] Mail-Konto für automatischen Rechnungseingang
- [ ] Kurzbefehl "iOS-Scan → Paperless-API"
- [ ] Entscheiden, ob Tika/Gotenberg gebraucht wird (Office-Dokumente, Mails)
- [ ] `document_exporter` als regelmäßigen Cron-Job einrichten, mit `--passphrase`

---

## Änderungslog

| Datum | Änderung |
|---|---|
| 16.08.2026 | Erstfassung. Architektur, Verzeichnisaufteilung, Konfiguration, Export/Import-Ablauf und die Secret-Key-Lehre aus dem Testlauf dokumentiert. |
| 16.08.2026 | Abschnitt 8 ergänzt: Pflichtschritte nach dem Import (reindex, thumbnails, sanity check) und die 404-Diagnose — Ursache waren die kaputten Mail-Konten, nicht der Index. |
