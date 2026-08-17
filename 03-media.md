# Media-Stack (Servarr + Jellyfin)

**Angelegt:** 17.08.2026 · **Status:** geplant, nicht installiert
Gehört zu `00-fundament.md`. Umsetzung erst nach Schritt 5 (Restore-Test
auf echter Hardware).

**Vorlage:** [TechHutTV/homelab, Ordner `media`](https://github.com/TechHutTV/homelab/tree/main/media)
— gute Doku, aber für Proxmox mit Debian-VM geschrieben. Die Abweichungen
für Unraid stehen in Abschnitt 3.

---

## 1. Was der Stack macht

Neun Container, die zusammen eine Pipeline bilden:

```
Seerr ──► Sonarr / Radarr / Lidarr ──► Prowlarr ──► Indexer
(Wunsch)   (Verwaltung)                (Suche)      (Quellen)
                  │                        │
                  │                   FlareSolverr
                  │                   (Cloudflare)
                  ▼
           qBittorrent / NZBGet ──► /data/downloads
           (Download, im VPN)              │
                  │                        │ Hardlink
                  ▼                        ▼
              Bazarr              /data/filme, /data/serien
           (Untertitel)                    │
                                           ▼
                                       Jellyfin
                                       (Abspielen)
```

| Dienst | Rolle | Port |
|---|---|---|
| Gluetun | VPN-Tunnel + Kill-Switch für alles darunter | – |
| qBittorrent | Torrent-Client | 8080 |
| NZBGet | Usenet-Client | 6789 |
| Prowlarr | zentrale Indexer-Verwaltung, konfiguriert die *arr-Apps mit | 9696 |
| FlareSolverr | Headless-Browser, löst Cloudflare-Challenges für Indexer | 8191 |
| Sonarr | Serien | 8989 |
| Radarr | Filme | 7878 |
| Lidarr | Musik | 8686 |
| Bazarr | Untertitel, hängt an Sonarr/Radarr | 6767 |
| Seerr | Anfrage-Frontend für Mitbenutzer | 5055 |
| deunhealth | startet qBittorrent neu, wenn der VPN-Healthcheck kippt | – |
| Jellyfin | Media-Server | 8096 |

**Namen:** Es heißt **Prowlarr**. Und **Jellyseerr existiert nicht mehr** —
Overseerr und Jellyseerr sind im Februar 2026 zu **Seerr**
(`ghcr.io/seerr-team/seerr`) verschmolzen, beide Vorgänger sind deprecated
und die alten Instanzen wurden Ende März 2026 abgeschaltet. Seerr migriert
eine bestehende Config beim ersten Start automatisch.

---

## 2. Die Hardlink-Frage — der wichtigste Punkt

Radarr und Sonarr **verschieben** einen fertigen Download nicht, sie legen
einen **Hardlink** an: zwei Dateinamen, ein einziger Datenblock auf der
Platte. Vorteil: der Torrent kann weiterseeden, während die Datei
gleichzeitig sauber benannt in der Mediathek liegt — ohne doppelten
Speicherverbrauch und ohne Kopiervorgang.

Ein Hardlink funktioniert **nur innerhalb desselben Dateisystems.**
Klappt er nicht, fällt Radarr auf Kopieren zurück. Bei 40 GB pro Film ist
das der Unterschied zwischen "fertig in 2 Sekunden" und "10 Minuten
Plattenlast plus 40 GB doppelt".

### Konsequenz 1: ein Mount, nicht drei

Genau deshalb mountet die Vorlage überall stur `/data:/data` statt der
naheliegenden `/movies`, `/tv`, `/downloads`. Drei separate Mounts sehen
im Container aus wie drei Dateisysteme, auch wenn sie draußen dasselbe
sind — und der Hardlink scheitert.

**Regel: Alle Container bekommen genau einen Datenmount,
`/mnt/user/media:/data`.**

### Konsequenz 2: `downloads` gehört ins `media`-Share

`00-fundament.md`, Abschnitt 5, hat keinen `downloads`-Ordner. Er muss
rein — und zwar unter `media/`, nicht daneben:

```
/mnt/user/media/          ← ein Share, ein Mount
├── downloads/
│   ├── qbittorrent/{completed,incomplete,torrents}
│   └── nzbget/{completed,intermediate,nzb,queue,tmp}
├── filme/
├── serien/
├── musik/
└── fotos/                ← Immich, für diesen Stack irrelevant
```

Im Container heißt das dann `/data/downloads`, `/data/filme`, `/data/serien`.

Anlegen:

```bash
mkdir -p /mnt/user/media/downloads/qbittorrent/{completed,incomplete,torrents}
mkdir -p /mnt/user/media/downloads/nzbget/{completed,intermediate,nzb,queue,tmp}
chown -R 99:100 /mnt/user/media
```

### Konsequenz 3 (Unraid-spezifisch): gleiches Share reicht nicht ganz

Auf Unraid ist `/mnt/user` ein FUSE-Layer (`shfs`) über mehreren Platten.
Ein Hardlink kann keine Plattengrenze überschreiten. Liegt der Download
auf Disk 1 und Unraid platziert die Mediathek-Datei auf Disk 2, schlägt
`link()` mit `EXDEV` fehl und Radarr kopiert.

In der Praxis geht das meistens gut, weil beide Pfade im selben Share
liegen und die Allocation-Method Dateien auf derselben Platte hält —
**aber garantiert ist es nicht, und man merkt es nur an vollen Platten.**

→ **Nach dem ersten Import verifizieren:**

```bash
stat -c '%h  %n' "/mnt/user/media/filme/Irgendein Film (2024)/"*.mkv
```

Erste Spalte ist der Link-Count. **`2` = Hardlink hat funktioniert.**
`1` = es wurde kopiert, dann Allocation-Method und Split-Level des
Shares nachjustieren.

Das ist ein offener Punkt, kein gelöstes Problem (s. Abschnitt 7).

---

## 3. Abweichungen von der Vorlage

Fünf Stellen, an denen die TechHut-Config auf Unraid bricht:

### 3.1 PUID/PGID — 99:100 statt 1000:1000

Die Vorlage nutzt `1000:1000`, den Standard auf Debian/Ubuntu.
**Unraid nutzt `99:100`** (`nobody:users`). Ohne diese Änderung kann kein
Container in die Shares schreiben. Der mit Abstand häufigste Fehler.

### 3.2 Absolute Config-Pfade — sonst schreibt der Stack auf den USB-Stick

Die Vorlage nutzt relative Bind-Mounts (`./sonarr:/config`). Docker löst
die relativ zum Ort der Compose-Datei auf. Auf Unraid legt das
**Compose-Manager-Plugin** Projekte hier ab:

```
/boot/config/plugins/compose.manager/projects/<name>/
```

`/boot` ist der **USB-Boot-Stick**. Relative Pfade würden also SQLite-
Datenbanken auf einen USB-Stick schreiben — Dauerschreiblast auf Flash,
und das Flash-Backup (`00-fundament.md`, Abschnitt 6) bläht sich von
wenigen MB auf Gigabytes auf.

**Alle Config-Mounts absolut nach `/mnt/user/appdata/<dienst>`.**
Das ist dieselbe harte Regel wie in Abschnitt 5 des Fundaments: Datenbanken
auf den NVMe-Cache, nie ins Array und schon gar nicht auf den Stick.

### 3.3 CIFS/fstab entfällt komplett

Die Vorlage mountet die Daten per SMB von einem anderen Rechner. Bei uns
laufen Docker und Daten auf derselben Kiste — der ganze `cifs-utils`- und
`/etc/fstab`-Abschnitt ist gegenstandslos. Ersatzlos streichen.

### 3.4 Der LXC-TUN-Fix entfällt

`lxc.cgroup2.devices.allow` etc. brauchen nur Proxmox-LXCs.
`/dev/net/tun` gibt es auf Unraid nativ.

### 3.5 Jellyfin läuft nicht in der Test-VM

`devices: /dev/dri:/dev/dri` ist Intel Quick Sync. In der Unraid-VM auf
dem NUC gibt es kein iGPU-Passthrough, das Device fehlt, der Container
startet nicht. Dort auskommentieren — oder Jellyfin, sinnvoller, erst auf
der echten Kiste aufsetzen. Transcoding ist der Punkt, an dem Jellyfin
steht oder fällt; ohne Quick Sync testet man das Falsche.

### Was übernommen wird

Die Troubleshooting-Abschnitte der Vorlage sind gut und bleiben gültig:
Gluetun-Healthcheck (`HEALTH_VPN_DURATION_INITIAL=120s`), `deunhealth`
gegen hängendes qBittorrent, `tun0` als Netzwerk-Interface in qBittorrent,
`BLOCK_MALICIOUS=off` gegen den wachsenden unbound-DNS-Cache.

---

## 4. VPN — die eigentliche Auswahlentscheidung

### Warum das nicht optional ist

Bei Torrents veröffentlichst du deine IP-Adresse gegenüber jedem anderen
Teilnehmer im Schwarm — das ist keine Nebenwirkung, das ist das
Funktionsprinzip. In Deutschland ist genau das die Grundlage des
Abmahnwesens. Usenet hat dieses Problem strukturell nicht: dort lädst du
von einem Server, es gibt kein Verteilen.

Gluetun ist deshalb kein Komfort-Feature, sondern der **Kill-Switch**:
alle Downloader laufen über `network_mode: service:gluetun` und haben
damit gar kein eigenes Netzwerk-Interface. Bricht der Tunnel, sind sie
schlicht offline. Das ist die richtige Architektur — sie muss nur
verifiziert werden (Abschnitt 6).

### Port-Forwarding ist das Auswahlkriterium

Ohne einen vom VPN-Anbieter nach außen geöffneten Port bist du im Schwarm
nicht erreichbar: Downloads laufen langsamer, Uploads praktisch gar nicht,
und private Tracker werfen dich wegen schlechter Ratio raus.

**Das Feature ist selten geworden.** NordVPN und Surfshark hatten es nie.
Mullvad hat es 2023 abgeschafft, PIA und IVPN zwischen 2023 und 2025.

| Anbieter | Port-Forwarding | Gluetun | Anmerkung |
|---|---|---|---|
| **ProtonVPN Plus** | ja, dynamisch (NAT-PMP) | **nativ** (`VPN_PORT_FORWARDING=on`) | Gluetun holt den Port selbst und meldet ihn an qBittorrent — nichts von Hand |
| **AirVPN** | ja, statisch | über `FIREWALL_VPN_INPUT_PORTS` | Port einmal im Client Area anlegen, dann fest. Empfehlung der Vorlage |
| PrivateVPN | ja | nativ | kleiner Anbieter |
| TorGuard, OVPN | ja | manuell | |
| Mullvad, PIA, IVPN | **nein mehr** | – | nicht mehr geeignet |
| NordVPN, Surfshark | nein | – | nie geeignet |

**Meine Einschätzung:** ProtonVPN Plus, wenn du eine Lösung willst, die
sich selbst repariert — der dynamische Port wird nach jedem Reconnect neu
geholt und automatisch in qBittorrent gesetzt. AirVPN, wenn du lieber
einen festen Port hast, den du selbst kennst und in der `.env` stehen
siehst. Beides tragfähig; der Unterschied ist "weniger Handgriffe" gegen
"weniger Magie".

Bei **reinem Usenet** brauchst du keinen VPN für Downloads. Gluetun bleibt
trotzdem sinnvoll für Prowlarr, weil Indexer-Anfragen sonst von deiner
Heim-IP kommen.

---

## 5. Dateien

Beides liegt in `media-stack/`:

| Datei | Inhalt |
|---|---|
| `compose.yaml` | vollständiger Stack, Unraid-angepasst, kommentiert |
| `.env.example` | Vorlage — als `.env` kopieren und ausfüllen |

Die echte `.env` enthält den **privaten WireGuard-Key**. Sie gehört ins
Passwortmanager-Backup und in keine Cloud. Das ist derselbe Fall wie
`PAPERLESS_SECRET_KEY` (`02-paperless.md`, Abschnitt 7): ohne diesen
Schlüssel ist ein Restore unvollständig, und man merkt es erst hinterher.

Wichtig: Das Compose-Projekt lebt auf dem USB-Stick und ist damit Teil des
**Flash-Backups** — die `.env` also auch. Ein Flash-Backup in fremder Hand
enthält deinen VPN-Key. Beim Ablegen entsprechend behandeln.

---

## 6. Reihenfolge bei der Umsetzung

Nicht alles auf einmal. Nach jedem Schritt prüfen, ob er hält.

1. **Ordner anlegen und Rechte setzen** (Abschnitt 2).
2. **Nur Gluetun starten.** Nichts anderes.
3. **Kill-Switch verifizieren — vor dem ersten Download:**

   ```bash
   docker run --rm --network=container:gluetun alpine:3.18 \
     sh -c "apk add --no-cache wget && wget -qO- https://ipinfo.io"
   ```

   Muss die IP des VPN-Anbieters zeigen, nicht deine. Zeigt sie deine
   eigene IP oder einen Fehler: **hier aufhören und das erst reparieren.**
4. **qBittorrent dazu.** Passwort aus dem Log holen
   (`docker logs qbittorrent`), sofort ändern, Pfade setzen
   (`/data/downloads/qbittorrent/{completed,incomplete,torrents}`),
   Netzwerk-Interface in den erweiterten Einstellungen auf `tun0`.
5. **Nochmal aus dem Container heraus prüfen:**

   ```bash
   docker exec -it qbittorrent wget -qO- https://ipinfo.io
   ```
6. **Prowlarr**, Indexer eintragen. FlareSolverr unter
   _Settings → Indexers → Indexer Proxies_ als `http://localhost:8191`
   (gleicher Netzwerk-Namespace wie Gluetun).
7. **Radarr**, ein einzelner Testfilm. Danach **Link-Count prüfen**
   (Abschnitt 2). Erst wenn der `2` ist, weitermachen.
8. Sonarr, Lidarr, Bazarr, Seerr.
9. Jellyfin zuletzt, mit Quick Sync (`intel_gpu_top` zur Kontrolle).

---

## 7. Offene Punkte

- [ ] VPN-Anbieter wählen: ProtonVPN Plus (dynamisch) oder AirVPN (statisch)
- [ ] Download-Weg festlegen: Torrent, Usenet oder beides.
      Bei reinem Usenet fliegen qBittorrent und deunhealth raus,
      dafür kommen Provider- und Indexer-Kosten (~50–100 €/Jahr)
- [ ] **Hardlink-Verhalten auf dem echten Array verifizieren** —
      Allocation-Method und Split-Level des `media`-Shares so setzen,
      dass `downloads` und Mediathek auf derselben Platte landen
- [ ] Prüfen, ob Port 8080 auf der echten Kiste frei ist
- [ ] Jellyfin-Zugriff von unterwegs: über Tailscale, nicht per
      Portfreigabe — analog zur Paperless-Entscheidung
      (`00-fundament.md`, Abschnitt 7)
- [ ] Entscheiden, ob `media/downloads` per Share-Einstellung vom
      Offsite-Backup ausgenommen wird (unnötiges Volumen)

---

## 8. Was in `00-fundament.md` angepasst werden muss

1. **Abschnitt 5 (Ordnerstruktur)** — `media/downloads/` mit den
   Unterordnern `qbittorrent/` und `nzbget/` ergänzen, plus einem Satz
   zur Begründung: gleiches Share ist Voraussetzung für Hardlinks.

2. **Abschnitt 7 (Reihenfolge)** — der Media-Stack fehlt. Er gehört als
   neuer Punkt zwischen 8 (Backup-Kette scharf) und 9 (qmd/Ollama), mit
   Verweis auf diese Datei. Bewusst *nach* der Backup-Kette: der Stack
   erzeugt schnell viele Terabyte, und die Backup-Entscheidung
   (was wird gesichert, was nicht) will vorher getroffen sein.

3. **Abschnitt 9 (Offene Punkte)** — VPN-Anbieter mit Port-Forwarding
   als neuer Punkt.

4. **Änderungslog** — Eintrag 17.08.2026.

---

## Änderungslog

| Datum | Änderung |
|---|---|
| 17.08.2026 | Erstfassung. TechHut-Vorlage gesichtet und für Unraid angepasst (PUID 99:100, absolute appdata-Pfade, CIFS/LXC-Teile gestrichen). Hardlink-Problematik als zentraler Planungspunkt dokumentiert. Jellyseerr → Seerr korrigiert. VPN-Anbieter recherchiert, noch nicht entschieden. |
