# HomeLAB

Planung und Dokumentation eines Homelabs auf Unraid-Basis. Langfristiges
Ziel ist ein lokaler Sprachassistent („Jarvis") als steuerndes Gehirn.

Dieses Repo ist die **einzige Quelle der Wahrheit** für Entscheidungen und
Konfiguration. Was hier nicht steht, ist nicht entschieden.

---

## Einstieg

**→ [`00-fundament.md`](00-fundament.md)** — Zielbild, Entscheidungen,
Hardware, Backup-Strategie. Alles andere hängt daran.

## Dateien

| Datei | Inhalt |
|---|---|
| [`00-fundament.md`](00-fundament.md) | Grundlage. Ziel, Entscheidungen samt Begründung, Hardware, Ordnerstruktur, Backup, Reihenfolge |
| [`01-testumgebung.md`](01-testumgebung.md) | Unraid-Test-VM auf Proxmox. Stick-GUID, VM-Parameter, Testplan |
| [`02-paperless.md`](02-paperless.md) | Paperless-ngx. Architektur, Konfiguration, Export/Import, Backup |
| [`03-media.md`](03-media.md) | Servarr-Stack und Jellyfin. Geplant, nicht installiert |
| `media-stack/` | Lauffähige Konfiguration zu `03-media.md` |

## Nummerierung

`00` ist die Grundlage, alles weitere sind Themen in der Reihenfolge, in
der sie entstanden sind — **nicht** in der Reihenfolge der Umsetzung.
Die steht in `00-fundament.md`, Abschnitt 7.

Kein Unterordner, solange es nicht wehtut. Vier Dateien brauchen keine
Verzeichnisstruktur.

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

- `media-stack/.env` — WireGuard-Private-Key
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
