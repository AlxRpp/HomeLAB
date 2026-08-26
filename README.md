# HomeLAB

Ein Homelab auf Unraid-Basis. Langfristiges Ziel ist ein lokaler
Sprachassistent („Jarvis“) als steuerndes Gehirn.

Das Repo ist zweierlei. **Planung** für das, was noch nicht steht: Server,
Speicher, Backup-Kette. Und **Dokumentation eines laufenden Systems** für
das, was seit September 2025 in Betrieb ist: die Hausautomation mit rund 200
Geräten, Sensor-Zeitreihen und einer Modellanbindung über MCP.

Dieses Repo ist die **einzige Quelle der Wahrheit** für Entscheidungen und
Konfiguration. Was hier nicht steht, ist nicht entschieden.

---

## Einstieg

**→ [`00-fundament.md`](00-fundament.md)** — Zielbild, Entscheidungen,
Hardware, Backup-Strategie. Alles andere hängt daran.

**→ [`04-hausautomation.md`](04-hausautomation.md)** — was heute läuft.
Geräte je Protokoll, Zeitreihen, Automationen samt der Fehlversuche
dahinter, MCP-Anbindung, offene Punkte.

## Dateien

Sortiert nach Relevanz, nicht nach Nummer.

| Datei | Inhalt |
|---|---|
| [`00-fundament.md`](00-fundament.md) | Grundlage. Ziel, Entscheidungen samt Begründung, Hardware, Ordnerstruktur, Backup, Reihenfolge |
| [`04-hausautomation.md`](04-hausautomation.md) | Home Assistant, in Betrieb. Geräte je Protokoll, Recorder und Zeitreihen, Automationen mit Begründung, MCP-Anbindung, offene Punkte |
| [`01-testumgebung.md`](01-testumgebung.md) | Unraid-Test-VM auf Proxmox. Stick-GUID, VM-Parameter, Testplan |
| [`02-paperless.md`](02-paperless.md) | Paperless-ngx. Architektur, Konfiguration, Export/Import, Backup |

### Weiteres

| Datei | Inhalt |
|---|---|
| [`media/03-media.md`](media/03-media.md) | Servarr-Stack und Jellyfin. Geplant, nicht installiert |
| `media/media-stack/` | Lauffähige Konfiguration zu `03-media.md` |

## Nummerierung

`00` ist die Grundlage, alles weitere sind Themen in der Reihenfolge, in
der sie entstanden sind — **nicht** in der Reihenfolge der Umsetzung.
Die steht in `00-fundament.md`, Abschnitt 7.

Bis August 2026 stand hier: kein Unterordner, solange es nicht wehtut.
Mit `media/` gibt es jetzt einen. Grund: Dort gehören eine Datei und ein
Konfigurationsverzeichnis zusammen, und beides ist ein Randthema. Für die
durchnummerierten Dateien bleibt es bei flacher Ablage.

---

## Drei Arten von Inhalt

Nützlich beim Schreiben und beim Wiederfinden — die drei verhalten sich
unterschiedlich:

| Typ | Verhalten | Beispiel |
|---|---|---|
| **Entscheidung** | wird nie editiert, nur ersetzt. Die alte Begründung bleibt lesbar | „Unraid statt Proxmox" |
| **Zustand** | wird überschrieben. Dafür ist die Versionskontrolle da | „Test-VM auf 192.168.178.156" |
| **Anleitung** | wird verbessert | „So exportierst du Paperless" |

Jede Entscheidung trägt ein Datum und eine Zeile **„Umwerfen würde das:"**.
Ohne die merkt man nie, wann eine Begründung nicht mehr trägt.

---

## Geheimnisse

`.gitignore` schließt `.env`, `*.conf`, Schlüssel und Paperless-Exporte
aus. **Vor jedem Commit prüfen:**

```bash
git status
git diff --cached          # vor dem Commit: was geht wirklich raus?
```

Ein einmal gepushter Key ist kompromittiert — dann hilft nur Rotieren,
nicht Löschen.

Was **nicht** ins Repo gehört, aber trotzdem gesichert sein muss
(→ `00-fundament.md`, Abschnitt 6, „Schlüssel sind Teil des Backups"):

- `media/media-stack/.env` — WireGuard-Private-Key
- `PAPERLESS_SECRET_KEY`
- Unraid-Flash-Backup

Diese drei gehören ins Passwortmanager-Backup, nicht hierher.

---

## Konventionen

**Commits** in ganzen Sätzen, deutsch, im Imperativ — beschreiben *warum*,
nicht *was*. Das Was steht im Diff.

```
Cache-Pool auf 2× 2 TB erhöht

Fotos und Dokumente ziehen vom Array auf den ZFS-Mirror, damit sie
Prüfsummen und Snapshots bekommen. Das Array wird dadurch reines
Media-Ziel und kann dauerhaft schlafen.
```

**Änderungslog:** Jede Datei hat unten eine Tabelle. Die bleibt auch mit
Git sinnvoll — sie erzählt die inhaltliche Entwicklung, während die
Git-History die Dateiänderungen erzählt.

**Korrigieren statt löschen:** Wird eine Entscheidung hinfällig, bleibt sie
mit einem Vermerk stehen. Sonst steht in zwei Jahren die neue Entscheidung
da, ohne dass jemand weiß, was vorher schiefging.
