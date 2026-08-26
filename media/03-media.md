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
stat -c '%h  %n' /mnt/user/media/filme/*/*
```

Erste Spalte ist der Link-Count. **`2` = Hardlink hat funktioniert.**
`1` = es wurde kopiert, dann Allocation-Method und Split-Level des
Shares nachjustieren.

**Ergebnis 17.08.2026:** Link-Count `2`. Aber siehe die nächste
Konsequenz — dieser Test war irreführend.

### Konsequenz 4: Der `media`-Share darf NICHT auf den Cache

Am 17.08.2026 fast übersehen, und der wichtigste Fund des Tages.

Unraid legt neue Shares standardmäßig mit *Primary storage: Cache* an.
Downloads landen dann zuerst auf dem Cache-Pool, und der **Mover**
schiebt sie nachts aufs Array. Klingt harmlos, ist es nicht:

- **Der Hardlink überlebt den Mover nicht.** Radarr verlinkt, solange
  beide Dateien auf dem Cache liegen. Verschiebt der Mover die
  Download-Datei später aufs Array, liegen die beiden auf
  verschiedenen Dateisystemen — der Link zerfällt zu zwei
  unabhängigen Kopien. **Ab da liegt jeder Film doppelt**, ohne
  Fehlermeldung.
- **Der Cache läuft voll.** In der Test-VM: 12 GB Downloads plus 7 GB
  `docker.img` auf einem 21-GB-Cache = 0 Byte frei.
- **Und dann fällt alles um, was auf dem Cache lebt.** Jellyfin konnte
  nicht mehr in seine SQLite-Datenbank schreiben:
  `Microsoft.EntityFrameworkCore.Update: An exception occurred in the
  database while saving changes`. Sieht aus wie ein Jellyfin-Bug, ist
  eine volle Platte.

**Der Hardlink-Test oben war deshalb ein Fehlalarm:** Er zeigte `2`,
weil Download *und* Mediathek in dem Moment beide auf dem Cache lagen.
Bestanden hätte er auch dann, wenn die Konfiguration falsch ist.

→ **Shares → media → Primary storage: `Array`, Secondary storage: `None`.**

Reihenfolge beim Umstellen eines bereits vollen Caches — die zählt:

```bash
# 1. Container stoppen, damit nichts in die Dateien schreibt
docker compose down

# 2. Mover laufen lassen, SOLANGE die Einstellung noch Cache → Array ist
mover start
watch -n 5 'df -h /mnt/cache | tail -1'

# 3. ERST DANACH: Primary storage auf Array, Secondary auf None
# 4. docker compose up -d
```

Stellt man zuerst um, weiß der Mover nicht mehr, dass er etwas
wegräumen soll, und die Daten bleiben auf dem Cache liegen.

**Nicht verwechseln mit der Regel aus `00-fundament.md`, Abschnitt 5.**
Dort steht: `appdata` gehört auf den Cache. Das stimmt weiterhin —
Datenbanken ja, Massendaten nein. `media` ist Massendaten.

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

### `SERVER_COUNTRIES` ist Pflicht, nicht optional

Am 17.08.2026 eine Stunde gekostet. **Proton erlaubt P2P nur in Ländern
mit lockerem Urheberrecht** — Niederlande, Schweiz, Island, Schweden,
Singapur. Auf allen anderen Servern, etwa UK, Deutschland oder USA, wird
BitTorrent-Verkehr blockiert.

`PORT_FORWARD_ONLY=on` reicht **nicht**. Das filtert auf
Port-Forwarding-Fähigkeit, nicht auf P2P-Erlaubnis — das sind
unterschiedliche Servermengen. Ohne `SERVER_COUNTRIES` landet man
irgendwo, im konkreten Fall in London.

**Das Fehlerbild ist tückisch, weil es wie ein Netzwerkproblem aussieht:**

- Der Tunnel steht, Healthcheck grün
- HTTP durch den Tunnel funktioniert einwandfrei
- Tracker-Anmeldungen gehen durch — der ipleak-Torrent-Test besteht sogar
- Nur die Peer-Verbindungen sterben nach wenigen Bytes, Status `Stalled`

Man sucht dann bei MTU, Ports und Firewall, und nichts davon ist es. Es
ist eine Richtlinie des Anbieters.

```bash
SERVER_COUNTRIES=Netherlands
```

---

## 5. VPN-Tunnel prüfen — Runbook

Der wichtigste Abschnitt dieser Datei. **Kein Downloader läuft, bevor
diese Tests durch sind.**

### Das Prinzip

qBittorrent und NZBGet haben über `network_mode: service:gluetun`
**kein eigenes Netzwerk-Interface**. Sie leihen sich den Netzwerk-Namespace
von gluetun. Es existiert physisch keine zweite Route nach draußen.

Das ist stärker als ein VPN-Client mit Kill-Switch-Häkchen: dort ist der
Kill-Switch eine Firewall-Regel, die greifen *soll*. Hier gibt es
schlicht kein Netzwerk, wenn der Tunnel weg ist.

Trotzdem prüfen. Vertrauen ist keine Verifikation.

### Welche IP darf herauskommen?

| IP | Bedeutung |
|---|---|
| Eine ProtonVPN-IP | richtig — der Tunnel trägt |
| **Deine Heim-IP** | **Alarm. Sofort alles stoppen.** |
| Timeout / keine Antwort | richtig, wenn der Tunnel absichtlich unten ist |

**Die Heim-IP nicht auswendig lernen.** Sie ändert sich bei den meisten
Anschlüssen täglich. Die belastbare Regel ist ein **Vergleich**: Was der
Unraid-Host sieht, darf nie identisch sein mit dem, was ein Container im
gluetun-Namespace sieht.

### Schnell-Check

Der eine Befehl, den man sich merkt:

```bash
docker run --rm --network=container:gluetun alpine \
  wget -qO- --timeout=8 https://1.1.1.1/cdn-cgi/trace | grep ^ip=
```

`1.1.1.1` ist bewusst eine **IP-Adresse, kein Name** — der Test kommt
damit ohne DNS aus. Und `/cdn-cgi/trace` gibt die öffentliche IP des
Aufrufers zurück, man braucht also keinen zweiten Dienst.

Als Vergleich mit automatischer Bewertung:

```bash
HOST_IP=$(curl -s https://ipinfo.io/ip)
VPN_IP=$(docker run --rm --network=container:gluetun alpine \
  wget -qO- --timeout=8 https://1.1.1.1/cdn-cgi/trace | grep ^ip= | cut -d= -f2)
echo "Host: $HOST_IP"
echo "VPN:  $VPN_IP"
[ "$HOST_IP" = "$VPN_IP" ] && echo ">>> ALARM: identisch!" || echo ">>> OK: unterschiedlich"
```

### Die vier Tests

**Test 1 — Grundlage: unterscheiden sich Host und Tunnel?**

```bash
curl -s https://ipinfo.io/ip                    # Unraid-Host
docker logs gluetun | grep "Public IP address"  # im Tunnel
```

Erwartung: zwei verschiedene IPs. Sind sie gleich, geht gar nichts durch
den Tunnel.

---

**Test 2 — gilt das auch für andere Container?**

```bash
docker run --rm --network=container:gluetun alpine wget -qO- https://ipinfo.io/ip
```

Erwartung: die VPN-IP. Das ist der eigentlich relevante Test, denn genau
so hängen qBittorrent und NZBGet später am Tunnel — als fremde Container
im selben Namespace.

---

**Test 3 — Tunnel bricht, Container lebt weiter.**

Der realistische Störfall: WLAN zuckt, Proton-Server überlastet,
Reconnect. Der Container läuft weiter, nur der Tunnel ist tot. Schlecht
gebaute Setups laden hier still über die Heim-IP weiter.

```bash
docker exec gluetun ip link set tun0 down && \
docker run --rm --network=container:gluetun alpine \
  wget -qO- --timeout=8 https://1.1.1.1/cdn-cgi/trace; echo " EXIT: $?"
```

Erwartung: `wget: download timed out`, Exit `1`.

Alles in **einer** Zeile, weil gluetun den Tunnel nach wenigen Sekunden
selbst wieder hochzieht (`Restart VPN on healthcheck failure: yes`).

> **Warum `1.1.1.1` und nicht `ipinfo.io`?**
> Mit einem Namen schlägt der Test schon an der DNS-Auflösung fehl
> (`wget: bad address`). Das beweist nur, dass DNS tot ist — nicht, dass
> keine Route existiert. Erst der direkte Zugriff auf eine IP-Adresse
> zeigt, dass es **überhaupt keinen Weg nach draußen** gibt.

---

**Test 4 — repariert er sich selbst?**

```bash
sleep 45; docker run --rm --network=container:gluetun alpine \
  wget -qO- --timeout=8 https://1.1.1.1/cdn-cgi/trace | grep ^ip=
```

Erwartung: wieder eine Proton-IP. Oft eine **andere** als vorher, weil
gluetun sich einen neuen Server sucht. Das ist gewollt — und bedeutet
gleichzeitig, dass der weitergeleitete Port ein anderer ist.

---

**Test 5 — der echte Torrent-Test.**

Die Tests 1–4 prüfen HTTP. BitTorrent meldet sich bei Trackern über
andere Ports und Protokolle — ohne diesen Test ist der Tunnel für
Torrent-Verkehr **nicht verifiziert**.

Auf `ipleak.net` im Bereich *Torrent Address detection* den Magnet-Link
holen und in qBittorrent laden. ipleak betreibt einen eigenen Tracker und
protokolliert, von welcher IP sich der Client anmeldet. Nach ein paar
Sekunden erscheint sie auf der Seite. Es wird nichts heruntergeladen, der
Torrent ist eine Attrappe.

> **Zwei IPs, nicht verwechseln.** Ganz oben zeigt die Seite die IP deines
> **Browsers** — also des Macs, der nicht über VPN läuft. Die ist richtig
> und darf deine Heim-IP sein. Nur die Zeile unter *Torrent Address
> detection* zählt, das ist qBittorrent im Container.

### Ergebnis vom 17.08.2026

Getestet in der Unraid-Test-VM, gluetun mit ProtonVPN/WireGuard.

| Test | Ergebnis |
|---|---|
| 1 — Host vs. Tunnel | bestanden: Host `92.208.x.x`, Tunnel `79.135.105.206` (Marseille) |
| 2 — fremder Container | bestanden: `79.135.105.206` |
| 3 — Tunnel unten | bestanden: Timeout, Exit 1, **kein Leck** |
| 4 — Selbstheilung | bestanden: nach ~45 s wieder da, neuer Server |
| 5 — Torrent | bestanden: ipleak.net meldet `217.138.216.105`, also die Proton-IP |

Ebenfalls bestätigt: `port forwarded is 51773` — NAT-PMP funktioniert,
der Proton-Schlüssel wurde also korrekt mit aktiviertem Port-Forwarding
erzeugt.

### Was die Tests nicht beweisen

Ehrlichkeit über die Grenzen, sonst wiegt das Runbook in falscher Sicherheit:

- **Torrent-Verkehr** — siehe Test 5, steht aus.
- **Reboot-Verhalten** — `depends_on` gilt nur bei `docker compose up`,
  nicht bei Dockers eigener Restart-Policy nach einem Neustart des Hosts.
  Muss separat geprüft werden.
- **Dauerbetrieb** — eine Messung beweist nichts über drei Monate.

### qBittorrent NICHT an `tun0` binden

Viele Anleitungen — auch die TechHut-Vorlage — empfehlen, in qBittorrent
unter *Erweitert* die **Netzwerkschnittstelle auf `tun0`** zu setzen.

**In diesem Setup ist das falsch.** Es hilft nicht und es schadet:

- **Redundant:** Der Container hat über `network_mode: service:gluetun`
  gar kein eigenes Netzwerk. Es gibt keine zweite Route, an die er sich
  binden könnte. Der Schutz sitzt im geteilten Namespace.
- **Schädlich:** Die Bindung stört libtorrent beim Aufbau der
  DHT-Sockets. Symptom: **`DHT: 0 nodes`**, dauerhaft, über
  Serverwechsel hinweg. Damit fällt die halbe Peer-Findung aus, es
  bleibt bei ein bis zwei Verbindungen, und der Download kriecht.

Der Rat stammt aus Setups, in denen der Client ein **eigenes** Netzwerk
hat und der VPN nur eine zusätzliche Route ist. Dort ist die Bindung
sinnvoll. Hier nicht.

→ *Optionen → Erweitert → Netzwerkschnittstelle:* **„Any interface"**

Nach dem Umstellen **Test 3 erneut fahren** — am 17.08.2026 geprüft, der
Kill-Switch hält ohne die Bindung unverändert (Timeout, Exit 1).

### Wann wiederholen

- nach jeder Änderung an `compose.yaml` oder `.env`
- nach `docker compose pull` (neues gluetun-Image)
- nach jedem Neustart der Unraid-Maschine
- nach einem Wechsel des VPN-Anbieters oder -Servers
- sonst etwa monatlich, mindestens der Schnell-Check

---

## 6. Die Dienste miteinander verbinden

Am 17.08.2026 einmal komplett durchgespielt. Die Stolpersteine stehen
hier, damit der zweite Aufbau auf der echten Maschine eine Stunde statt
eines Tages dauert.

### Adressen: warum feste IPs statt Containernamen

Der Stack zerfällt in **zwei Netzwerkwelten**:

| Welt | Dienste | Adresse |
|---|---|---|
| Im gluetun-Namespace | gluetun, qBittorrent, NZBGet, Prowlarr, FlareSolverr | alle unter `172.39.0.2` |
| Eigene Container | Radarr, Sonarr, Lidarr, Bazarr, Seerr, Jellyfin | eigene IP aus der `.env` |

Container im gluetun-Namespace benutzen **gluetuns DNS-Resolver**, der
nach außen zu Cloudflare geht — der kennt keine Docker-Containernamen.
`http://radarr:7878` funktioniert von Prowlarr aus also **nicht**.
Deshalb vergibt die compose feste IPs. Genau dafür sind sie da.

Und qBittorrent hat kein eigenes Netzwerk: Seine WebUI hängt an gluetuns
Adresse. In Radarr heißt der Download-Client-Host deshalb `gluetun`,
nicht `localhost`.

### Was wohin eingetragen wird

| In | Feld | Wert |
|---|---|---|
| Radarr | Download Client → Host | `gluetun`, Port `8080` |
| Radarr | Download Client → Category | `radarr` |
| Radarr | Root Folder | `/data/filme` |
| Prowlarr | Apps → Prowlarr Server | `http://172.39.0.2:9696` |
| Prowlarr | Apps → Radarr Server | `http://172.39.0.4:7878` |
| Seerr | Jellyfin URL | `172.39.0.8`, Port `8096` |
| Seerr | Radarr Server | `172.39.0.4`, Port `7878` |

API-Schlüssel stehen jeweils unter *Settings → General*. Jede *arr-App
hat genau einen, und den benutzen alle, die mit ihr reden.

### Fallstricke

**Prowlarr-Tags.** Ein Indexer mit Tag wird **nur** an Apps mit
demselben Tag weitergereicht. Wer einem Indexer zum Testen einen
`flaresolverr`-Tag verpasst, wundert sich später, warum er in Radarr
fehlt. Tag wieder entfernen, sobald er nicht gebraucht wird.

**Öffentliche Indexer sind unzuverlässig.** Ihre Definitionen in
Prowlarr hinken den ständigen Domain- und Strukturwechseln hinterher.
Von den getesteten liefen etwa die Hälfte. `403 Forbidden` bedeutet
meist nicht „VPN blockiert", sondern „Definition veraltet" — prüfbar
mit einem direkten Abruf aus dem Container heraus:

```bash
docker exec gluetun wget -qO- --timeout=10 https://<indexer-domain>/ | head -20
```

Kommt normales HTML, ist die Seite erreichbar und der Fehler liegt in
der Definition. FlareSolverr hilft dann nicht.

**Jellyfin merkt neue Dateien nicht.** Die „Echtzeitüberwachung"
stützt sich auf inotify, und das funktioniert auf Unraids
FUSE-Layer `/mnt/user` nicht zuverlässig. Lösung ist nicht häufigeres
Scannen, sondern eine Benachrichtigung:

→ *Radarr → Settings → Connect → + → Emby/Jellyfin*, Host `172.39.0.8`,
Port `8096`, Jellyfin-API-Schlüssel (aus *Dashboard → Erweitert →
API-Schlüssel*), **Update Library** an, Trigger *On Import* und
*On Upgrade*.

**Seerr ignoriert PUID/PGID.** Läuft fest als `node` = UID 1000. Der
appdata-Ordner braucht `chown -R 1000:1000`, sonst Startschleife mit
`EACCES: permission denied, mkdir '/app/config/logs/'`. Einzige
Ausnahme im ganzen Stack.

**Seerr-Login = Jellyfin-Login.** Es gibt kein eigenes Passwort.

### `Minimum Availability: Released` — nicht optional

Steuert, ab wann Radarr überhaupt zu suchen beginnt:

| Wert | Verhalten |
|---|---|
| `Announced` | sucht ab Ankündigung |
| `In Cinemas` | sucht ab Kinostart |
| `Released` | sucht ab digitaler Veröffentlichung |

Bei den ersten beiden holt Radarr für einen Film, der noch im Kino
läuft, zwangsläufig eine **Kino-Abfilmung** — es gibt nichts anderes.
Am 17.08.2026 genau so passiert: eingebrannte Untertitel, unscharfes
Bild, und die Datei hieß trotzdem `WEBDL-1080p`.

Denn: Qualitätsprofile helfen nur gegen *ehrlich* benannte Releases.
`CAM`, `TELESYNC`, `TELECINE`, `WORKPRINT` und `DVDSCR` sollte man im
Profil abwählen — gegen bewusste Falschbenennung kann kein Werkzeug
etwas ausrichten, weil niemand in die Datei hineinschaut. Der
zuverlässige Schutz ist `Released`.

### Dateibenennung für Jellyfin

Radarr benennt nur um, wenn *Settings → Media Management →
**Rename Movies*** aktiv ist. Standardmäßig ist es das nicht.

```
Standard Movie Format:  {Movie CleanTitle} ({Release Year}) {Quality Full}
Movie Folder Format:    {Movie CleanTitle} ({Release Year}) [imdbid-{ImdbId}]
```

Das Feld für den Ordner erscheint erst über **Show Advanced**.

Die **IMDb-ID im Ordnernamen** ist der Aufwand wert: Jellyfin muss dann
nicht mehr raten und trifft auch bei deutschen Verleihtiteln, die vom
Original abweichen, die richtigen Metadaten.

Das Ordnerformat greift nur beim **Hinzufügen** eines Films. Bereits
vorhandene behalten ihren Ordnernamen, bis man sie über den
Film-Editor verschiebt.

---

## 7. Dateien

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

## 8. Reihenfolge bei der Umsetzung

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

## 9. Offene Punkte

- [x] ~~VPN-Anbieter wählen~~ — **ProtonVPN**, 17.08.2026. WireGuard,
      `VPN_PORT_FORWARDING=on` + `PORT_FORWARD_ONLY=on`. Getestet, siehe Abschnitt 5
- [x] ~~Test 5 (Torrent-IP)~~ — bestanden am 17.08.2026
- [x] ~~`HEALTH_VPN_DURATION_INITIAL` entfernen~~ — erledigt, war obsolet
- [ ] Reboot-Verhalten prüfen: kommt qBittorrent nach einem Neustart der
      Unraid-Maschine wirklich erst nach gluetun hoch? `depends_on` gilt
      nur bei `docker compose up`, nicht bei Dockers Restart-Policy
- [ ] MTU: gluetun setzt in der Test-VM 999 statt der üblichen ~1420.
      Auf der echten Hardware erneut prüfen, kostet sonst Durchsatz
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

## 10. Was in `00-fundament.md` angepasst werden muss

1. **Abschnitt 5 (Ordnerstruktur)** — `media/downloads/` mit den
   Unterordnern `qbittorrent/` und `nzbget/` ergänzen, plus einem Satz
   zur Begründung: gleiches Share ist Voraussetzung für Hardlinks.

2. **Abschnitt 7 (Reihenfolge)** — der Media-Stack fehlt. Er gehört als
   neuer Punkt zwischen 8 (Backup-Kette scharf) und 9 (qmd/Ollama), mit
   Verweis auf diese Datei. Bewusst *nach* der Backup-Kette: der Stack
   erzeugt schnell viele Terabyte, und die Backup-Entscheidung
   (was wird gesichert, was nicht) will vorher getroffen sein.

   **Ergänzung 17.08.2026:** Die Regel in Abschnitt 5 lautet bisher
   „`appdata` gehört auf den Cache". Sie braucht die Umkehrung als
   eigenen Satz: **Massendaten-Shares gehören explizit auf `Array`,
   nicht auf Cache.** Unraid legt neue Shares standardmäßig mit
   *Primary storage: Cache* an — man muss aktiv umstellen. Sonst
   laufen Cache-Pool und damit auch alle Datenbanken darauf voll,
   und Hardlinks zerfallen still, sobald der Mover läuft
   (Begründung in Abschnitt 2, Konsequenz 4).

3. **Abschnitt 9 (Offene Punkte)** — VPN-Anbieter mit Port-Forwarding
   als neuer Punkt.

4. **Änderungslog** — Eintrag 17.08.2026.

---

## Änderungslog

| Datum | Änderung |
|---|---|
| 17.08.2026 | Erstfassung. TechHut-Vorlage gesichtet und für Unraid angepasst (PUID 99:100, absolute appdata-Pfade, CIFS/LXC-Teile gestrichen). Hardlink-Problematik als zentraler Planungspunkt dokumentiert. Jellyseerr → Seerr korrigiert. VPN-Anbieter recherchiert, noch nicht entschieden. |
| 17.08.2026 | Kompletter Durchlauf: Prowlarr, Radarr, Seerr, Jellyfin. Neuer Abschnitt 6 mit Verkabelung und Fallstricken. **Wichtigster Fund: `media` darf nicht auf den Cache** — sonst zerfallen Hardlinks beim Mover und der Cache läuft voll (Abschnitt 2, Konsequenz 4). Der Hardlink-Test vom Vormittag war dadurch ein Fehlalarm. |
| 17.08.2026 | qBittorrent aufgesetzt, Test 5 (Torrent-IP über ipleak.net) bestanden. Dynamisches Port-Forwarding über `VPN_PORT_FORWARDING_UP_COMMAND` eingerichtet — die Befehle stehen in `.env.example`, damit sie versioniert sind, da die echte `.env` bewusst nicht ins Repo geht. |
| 17.08.2026 | **ProtonVPN gewählt** (dynamischer Port via NAT-PMP, gluetun unterstützt das nativ). Gluetun in der Test-VM aufgesetzt, Tests 1–4 bestanden. Neuer Abschnitt 5 als Runbook, Rest neu nummeriert. `TORRENTING_PORT` aus der `compose.yaml` entfernt — bei Proton ist der Port dynamisch. |
