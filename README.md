# SRT Streaming Relay Server
## Installations- und Aufbauanleitung – Phase 1 bis 3

**Stand:** Oktober 2026  
**Zielgruppe:** Personen, die den Server Schritt für Schritt installieren möchten und nicht voraussetzen können, dass die einzelnen Komponenten bereits bekannt sind.

---

# Vorwort

Diese Anleitung beschreibt den Aufbau eines zentralen Streaming-Servers für einen Streamer.

Die Grundidee ist relativ einfach:

Der Streamer erzeugt auf seinem PC mit OBS bereits das fertige Video. Statt dieses fertige Video direkt zu Twitch und YouTube zu schicken, wird es zunächst an unseren eigenen Server gesendet.

Der Server nimmt den Stream entgegen und verteilt ihn anschließend weiter.

```text
OBS / Streamer-PC
        │
        │ SRT
        ▼
┌─────────────────────┐
│   Netcup Server     │
│                     │
│      MediaMTX       │
│                     │
└─────────┬───────────┘
          │
      ┌───┴────┐
      ▼        ▼
   Twitch   YouTube
```

Der große Vorteil dieser Architektur:

Der Streamer muss nur **eine Verbindung zum Server** aufbauen.

Der Server kann daraus mehrere Ausgänge erzeugen.

Zusätzlich kann der Server einen Stream-Ausfall erkennen. Wenn der Streamer beispielsweise kurz die Internetverbindung verliert, kann der Server anstelle des Live-Bildes ein vorbereitetes Fallback-Video abspielen:

```text
Streamer offline
       │
       ▼
   MediaMTX
       │
       ▼
 fallback.mp4
       │
       ├──► Twitch
       └──► YouTube
```

Sobald der Streamer wieder verbunden ist:

```text
Streamer wieder online
       │
       ▼
   MediaMTX
       │
       ▼
   Live-Stream
```

Dadurch müssen die Plattform-Ausgaben nicht zwangsläufig wegen eines kurzen Verbindungsabbruchs beendet werden.

---

# Warum wird der Stream nicht auf dem Server neu encodiert?

Der Stream von OBS ist bereits fertig encodiert.

Geplant ist beispielsweise:

```text
1920 × 1080
60 FPS
H.264
ca. 6000–6200 Kbit/s
AAC
```

Der Server soll dieses Video nicht erneut berechnen.

Er soll es hauptsächlich:

1. entgegennehmen,
2. verwalten,
3. vervielfältigen,
4. an die Plattformen weiterleiten.

Das spart CPU-Leistung und macht einen kleinen VPS ausreichend.

---

# Gesamtarchitektur

Das fertige System besteht aus mehreren logisch getrennten Komponenten.

```text
                         ┌──────────────► Twitch
                         │
OBS / PC ──SRT──────────►│
                         │
                         │ MediaMTX
                         │
                         └──────────────► YouTube

                              ▲
                              │
                         Controller
                              ▲
                 ┌────────────┴────────────┐
                 │                         │
            Twitch Chat              YouTube Chat
                 │                         │
                 └────────────┬────────────┘
                              │
                       Chat-Commands
```

Später kommt für IRL-Streaming zusätzlich SRTLA dazu:

```text
OBS / PC ──SRT───────┐
                     │
                     ▼
                  MediaMTX
                     │
IRL / Moblin ─SRTLA──┤
                     │
                ┌────┴────┐
                ▼         ▼
             Twitch    YouTube
```

---

# Aufbau der Anleitung

Die Installation wird in drei große Phasen aufgeteilt:

```text
PHASE 1
Server-Grundinstallation
        │
        ▼
PHASE 2
MediaMTX + SRT + Fallback
        │
        ▼
PHASE 3
Twitch + YouTube + Controller + Chat + Webinterface
```

Die Phasen sollten nacheinander abgeschlossen werden.

**Nicht mehrere Phasen gleichzeitig verändern.**

---

# 1. Technische Zielwerte

## 1.1 Stream

| Eigenschaft | Ziel |
|---|---|
| Auflösung | 1920 × 1080 |
| Framerate | 60 FPS |
| Video | H.264 / AVC |
| Video-Bitrate | ca. 6000–6200 Kbit/s |
| Bitrate-Modus | CBR |
| Keyframe-Intervall | 2 Sekunden |
| Audio | AAC |
| Audio-Bitrate | ca. 160 Kbit/s |
| Server-Transcoding | Nein |

## 1.2 Netzwerk

Bei ca. 6,2 Mbit/s Eingang:

```text
Server-Eingang:
≈ 6,2 Mbit/s

Twitch:
≈ 6,2 Mbit/s

YouTube:
≈ 6,2 Mbit/s
```

Bei beiden Plattformen gleichzeitig:

```text
Eingang:   ≈ 6,2 Mbit/s
Ausgang:   ≈ 12,4 Mbit/s
```

zuzüglich Protokoll-Overhead.

---

# 2. Verwendete Komponenten

## 2.1 Ubuntu

Betriebssystem des Servers.

## 2.2 Docker

Docker kapselt die einzelnen Dienste in Containern.

Das bedeutet vereinfacht:

```text
Server
 │
 ├── MediaMTX-Container
 │
 └── Controller-Container
```

Dadurch können die Komponenten unabhängig voneinander aktualisiert und neu gestartet werden.

## 2.3 MediaMTX

MediaMTX ist der eigentliche Streaming-Server.

Er übernimmt unter anderem:

- SRT-Eingang
- Stream-Pfade
- Weitergabe von Streams
- Fallback-Stream
- Protokollkonvertierung bzw. Weiterleitung

## 2.4 Controller

Der Controller ist eine separate Anwendung.

Er ist **nicht MediaMTX**.

Der Controller kümmert sich um:

- Twitch
- YouTube
- Chat
- Chat-Befehle
- Berechtigungen
- Status
- später Webinterface

Dadurch bleibt die Streaming-Funktion von der Steuerungslogik getrennt.

---

# 3. Phase 1 – Server-Grundinstallation

## Ziel

Nach Phase 1 existieren:

- Ubuntu Server 24.04 LTS
- ein eigener Administrator-Benutzer
- SSH-Key-Zugriff
- kein Root-SSH
- kein Passwort-SSH
- UFW
- Docker
- Docker Compose

---

## 3.1 VPS bereitstellen

Geplant:

**Netcup VPS 500 G12.5**

Der Server kann später erweitert werden.

```text
VPS 500
   │
   ▼
VPS 1000
   │
   ▼
ggf. später RS
```

Ein Wechsel von VPS auf RS ist kein einfacher Tarifwechsel. Ein RS wäre eine neue Instanz, auf die anschließend migriert werden müsste.

---

## 3.2 Erstmalige Verbindung

```bash
ssh root@SERVER_IP
```

`SERVER_IP` durch die öffentliche IP-Adresse des Servers ersetzen.

---

## 3.3 Server kontrollieren

```bash
hostnamectl
```

```bash
lsb_release -a
```

```bash
free -h
```

```bash
df -h
```

```bash
nproc
```

```bash
ip addr
```

---

## 3.4 System aktualisieren

```bash
apt update
```

```bash
apt upgrade -y
```

Neustart:

```bash
reboot
```

Danach erneut verbinden.

---

## 3.5 Administrator anlegen

```bash
adduser streamer
```

```bash
usermod -aG sudo streamer
```

Test:

```bash
su - streamer
```

```bash
sudo whoami
```

Erwartung:

```text
root
```

---

## 3.6 Weitere Administratoren

Jeder Administrator erhält ein eigenes Konto.

Beispiel:

```text
streamer
admin2
admin3
```

Kein gemeinsames Konto verwenden.

Vorteile:

- einzelne Zugänge können entfernt werden
- keine gemeinsam genutzten Passwörter
- bessere Nachvollziehbarkeit
- jeder Benutzer besitzt eigene SSH-Keys

---

## 3.7 SSH-Key erzeugen

Unter Windows PowerShell:

```powershell
ssh-keygen -t ed25519
```

Public Key anzeigen:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

---

## 3.8 Public Key installieren

Auf dem Server:

```bash
su - streamer
```

```bash
mkdir -p ~/.ssh
```

```bash
nano ~/.ssh/authorized_keys
```

Public Key einfügen.

Berechtigungen:

```bash
chmod 700 ~/.ssh
```

```bash
chmod 600 ~/.ssh/authorized_keys
```

---

## 3.9 SSH-Key testen

Eine zweite PowerShell öffnen:

```powershell
ssh streamer@SERVER_IP
```

Danach:

```bash
sudo whoami
```

Erwartet:

```text
root
```

**Die bisherige SSH-Sitzung noch nicht schließen.**

---

## 3.10 SSH absichern

```bash
sudo nano /etc/ssh/sshd_config
```

Setzen:

```text
PermitRootLogin no
PasswordAuthentication no
```

Prüfen:

```bash
sudo sshd -t
```

Wenn keine Fehlermeldung erscheint:

```bash
sudo systemctl restart ssh
```

Neue SSH-Verbindung testen:

```powershell
ssh streamer@SERVER_IP
```

Erst wenn das funktioniert, alte Root-Sitzung schließen.

---

## 3.11 UFW installieren

```bash
sudo apt install ufw -y
```

SSH freigeben:

```bash
sudo ufw allow OpenSSH
```

Aktivieren:

```bash
sudo ufw enable
```

Prüfen:

```bash
sudo ufw status verbose
```

---

## 3.12 Docker installieren

```bash
sudo apt install docker.io docker-compose-v2 -y
```

```bash
sudo systemctl enable docker
```

```bash
sudo systemctl start docker
```

Prüfen:

```bash
sudo docker --version
```

```bash
docker compose version
```

---

## 3.13 Docker ohne sudo

```bash
sudo usermod -aG docker streamer
```

SSH vollständig trennen und neu verbinden.

Test:

```bash
docker run hello-world
```

---

## 3.14 Phase-1-Kontrolle

- [ ] Ubuntu funktioniert
- [ ] System aktualisiert
- [ ] Admin-Benutzer vorhanden
- [ ] SSH-Key funktioniert
- [ ] Root-SSH deaktiviert
- [ ] Passwort-SSH deaktiviert
- [ ] UFW aktiv
- [ ] Docker funktioniert
- [ ] Docker Compose funktioniert

---

# 4. Phase 2 – MediaMTX + SRT + Fallback

## Ziel

Nach Phase 2:

```text
OBS
 │
 │ SRT
 ▼
MediaMTX
 │
 ├── live
 │
 └── fallback.mp4
```

Noch keine Twitch-/YouTube-Steuerung.

---

# 4.1 Verzeichnisstruktur

```bash
sudo mkdir -p /opt/streaming/mediamtx
```

```bash
sudo chown -R streamer:streamer /opt/streaming
```

```bash
cd /opt/streaming/mediamtx
```

---

# 4.2 MediaMTX-Konfiguration

```bash
nano /opt/streaming/mediamtx/mediamtx.yml
```

Inhalt:

```yaml
logLevel: info

srt: true
srtAddress: :8890

paths:
  live:
    source: publisher
```

MediaMTX verwendet UDP-Port 8890 für den SRT-Listener. Die aktuelle Konfigurationsreferenz führt `srtAddress: :8890` weiterhin als SRT-Serveradresse. citeturn0search3

---

# 4.3 Docker Compose

```bash
nano /opt/streaming/mediamtx/compose.yml
```

Inhalt:

```yaml
services:
  mediamtx:
    image: bluenviron/mediamtx:1
    container_name: mediamtx
    restart: unless-stopped

    network_mode: host

    volumes:
      - ./mediamtx.yml:/mediamtx.yml:ro
```

---

# 4.4 SRT-Port öffnen

```bash
sudo ufw allow 8890/udp
```

Prüfen:

```bash
sudo ufw status
```

---

# 4.5 MediaMTX starten

```bash
cd /opt/streaming/mediamtx
```

```bash
docker compose up -d
```

Prüfen:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs --tail=50
```

---

# 4.6 SRT-Port prüfen

```bash
sudo ss -lunp | grep 8890
```

Erwartet:

```text
UNCONN ... 0.0.0.0:8890 ...
```

---

# 4.7 OBS mit MediaMTX verbinden

OBS:

**Einstellungen → Stream → Benutzerdefiniert**

Server:

```text
srt://SERVER_IP:8890?streamid=publish:live
```

Für den ersten Test:

- kein SRT-Passwort
- Stream starten

MediaMTX-Logs:

```bash
docker compose logs -f
```

Erwartet:

```text
SRT-Verbindung
Publisher
Path: live
```

---

# 4.8 Video und Audio prüfen

```bash
docker compose logs --tail=100
```

Der Eingang soll enthalten:

```text
H264
AAC
```

Keine Transcodierung.

---

# 4.9 Verbindung trennen und wiederherstellen

OBS stoppen.

```bash
docker compose logs --tail=50
```

MediaMTX muss den Publisher-Verlust erkennen.

OBS erneut starten.

Die Verbindung muss wieder hergestellt werden.

---

# 4.10 Fallback-Verzeichnis

Jetzt wird der genaue Speicherort des Fallback-Videos festgelegt.

Auf dem Server:

```bash
cd /opt/streaming/mediamtx
```

```bash
mkdir -p fallback
```

Die Datei muss anschließend hier liegen:

```text
/opt/streaming/mediamtx/fallback/fallback.mp4
```

Also:

```text
/opt/
└── streaming/
    └── mediamtx/
        ├── compose.yml
        ├── mediamtx.yml
        └── fallback/
            └── fallback.mp4
```

---

# 4.11 Fallback-Datei auf den Server kopieren

Die Datei kann beispielsweise per SCP oder SFTP auf den Server übertragen werden.

Ziel:

```text
/opt/streaming/mediamtx/fallback/fallback.mp4
```

Prüfen:

```bash
ls -lh /opt/streaming/mediamtx/fallback/
```

Erwartet:

```text
fallback.mp4
```

---

# 4.12 Fallback-Video

Das Fallback sollte mit dem Live-Stream kompatible Codecs verwenden.

Empfohlen:

```text
Video:     H.264
Audio:     AAC
Auflösung: 1920x1080
FPS:       60
```

Das Video kann deutlich niedriger bitratig sein.

Beispielinhalt:

```text
Gleich geht's weiter ...
Der Stream wird fortgesetzt.
```

MediaMTX kann bei `alwaysAvailable` einen Offline-Abschnitt wiederholen und alternativ eine MP4-Datei verwenden. Der Wechsel erfolgt ohne erneutes Encodieren der Frames. citeturn0search0

---

# 4.13 Docker Compose für Fallback erweitern

```bash
nano /opt/streaming/mediamtx/compose.yml
```

Kompletter Inhalt:

```yaml
services:
  mediamtx:
    image: bluenviron/mediamtx:1
    container_name: mediamtx
    restart: unless-stopped

    network_mode: host

    volumes:
      - ./mediamtx.yml:/mediamtx.yml:ro
      - ./fallback:/fallback:ro
```

Der zweite Mount ist wichtig.

Er macht:

```text
Server:
/opt/streaming/mediamtx/fallback/fallback.mp4
```

im Container als:

```text
/fallback/fallback.mp4
```

sichtbar.

---

# 4.14 MediaMTX-Fallback konfigurieren

```bash
nano /opt/streaming/mediamtx/mediamtx.yml
```

Inhalt:

```yaml
logLevel: info

srt: true
srtAddress: :8890

paths:
  live:
    source: publisher
    alwaysAvailable: true
    alwaysAvailableFile: /fallback/fallback.mp4
```

`alwaysAvailableFile` muss auf den Pfad **im Container** zeigen, nicht auf den Host-Pfad. Genau deshalb wurde der Ordner in `compose.yml` nach `/fallback` gemountet. citeturn0search0turn0search3

---

# 4.15 Fallback-Datei im Container prüfen

```bash
docker exec mediamtx ls -lh /fallback/
```

Es muss `fallback.mp4` angezeigt werden.

Wenn nicht:

1. Dateipfad auf dem Host prüfen
2. Compose-Mount prüfen
3. Container neu erstellen

---

# 4.16 MediaMTX neu starten

```bash
cd /opt/streaming/mediamtx
```

```bash
docker compose down
```

```bash
docker compose up -d
```

Prüfen:

```bash
docker compose ps
```

```bash
docker compose logs --tail=50
```

---

# 4.17 Fallback testen

### Test 1

OBS starten.

```text
OBS
 ↓
SRT
 ↓
MediaMTX
 ↓
Live
```

### Test 2

OBS stoppen.

```text
OBS
 X
 ↓
MediaMTX
 ↓
fallback.mp4
```

### Test 3

OBS wieder starten.

```text
fallback.mp4
      ↓
     Live
```

---

# 4.18 Phase-2-Kontrolle

- [ ] MediaMTX läuft
- [ ] UDP 8890 geöffnet
- [ ] OBS verbindet per SRT
- [ ] Path `live` funktioniert
- [ ] H264 kommt an
- [ ] AAC kommt an
- [ ] Disconnect wird erkannt
- [ ] Reconnect funktioniert
- [ ] `/opt/streaming/mediamtx/fallback/fallback.mp4` existiert
- [ ] Fallback ist im Container sichtbar
- [ ] Fallback wird abgespielt
- [ ] Live übernimmt nach Reconnect wieder

**Erst jetzt Phase 3 beginnen.**

---

# 5. Phase 3 – Twitch, YouTube, Controller und Chat

## Ziel

Phase 3 fügt die eigentliche Steuerungslogik hinzu.

```text
                 ┌──────────► Twitch
                 │
OBS ─SRT─► MediaMTX
                 │
                 └──────────► YouTube
                       ▲
                       │
                  Controller
                       ▲
              ┌────────┴────────┐
              │                 │
         Twitch Chat       YouTube Chat
```

---

# 5.1 Warum gibt es einen Controller?

MediaMTX soll nicht für Chatbefehle zuständig sein.

MediaMTX ist der Streaming-Server.

Der Controller ist die Anwendung, die Entscheidungen trifft.

Beispiel:

```text
Chat:
!streamtwitch on
       │
       ▼
Controller
       │
       ▼
Twitch-Ausgabe starten
```

Oder:

```text
Chat:
!streamyoutube off
       │
       ▼
Controller
       │
       ▼
YouTube-Ausgabe stoppen
```

---

# 5.2 Controller-Verzeichnis erstellen

```bash
sudo mkdir -p /opt/streaming/controller/app
```

```bash
sudo chown -R streamer:streamer /opt/streaming/controller
```

Zielstruktur:

```text
/opt/streaming/
├── mediamtx/
│   ├── compose.yml
│   ├── mediamtx.yml
│   └── fallback/
│       └── fallback.mp4
│
└── controller/
    ├── compose.yml
    ├── app/
    │   ├── main.py
    │   ├── twitch.py
    │   ├── youtube.py
    │   └── ...
    └── .env
```

---

# 5.3 Chat-System – wichtige Unterscheidung

Der Server muss nicht einfach einen beliebigen „Port öffnen“ und dann Twitch-Chat empfangen.

Die Verbindung wird **ausgehend** vom Controller zu Twitch aufgebaut.

Das bedeutet:

```text
Controller
    │
    │ HTTPS / WebSocket / IRC
    ▼
Twitch
```

Es ist dafür **kein öffentlich erreichbarer IRC-Port auf dem Netcup-Server erforderlich**.

Das ist wichtig:

```text
Falsch:
Internet → Server: IRC-Port öffnen

Richtig:
Server → Twitch: Verbindung aufbauen
```

Für YouTube gilt ebenfalls:

```text
Controller
    │
    │ HTTPS / Streaming-Verbindung
    ▼
YouTube API
```

---

# 5.4 Twitch Chat – IRC oder EventSub?

Twitch unterstützt weiterhin IRC.

IRC kann Nachrichten über `PRIVMSG` an den Bot liefern und benötigt für das Lesen über IRC den Scope `chat:read`. Für das Senden über IRC wird `chat:edit` benötigt. citeturn1search0turn1search5

Twitch empfiehlt inzwischen für neue Chatbots allerdings EventSub zum Lesen von Chatnachrichten und die Twitch API zum Senden. citeturn1search1turn1search2

### Für dieses Projekt

Da der Chat-Listener nur wenige Steuerbefehle benötigt, gibt es zwei mögliche Varianten:

**Variante A – IRC**

```text
Controller
   │
   │ IRC
   ▼
Twitch Chat
```

Einfaches, direktes Chat-Listener-Modell.

**Variante B – EventSub**

```text
Controller
   │
   │ EventSub WebSocket
   ▼
Twitch Chat
```

Modernere Twitch-Variante und langfristig die bevorzugte Lösung.

Für eine neue produktive Implementierung sollte **EventSub bevorzugt** werden. Wenn ausdrücklich ein klassischer IRC-Listener gewünscht ist, kann dieser trotzdem umgesetzt werden.

---

# 5.5 Twitch-Chat mit IRC – Aufbau

Bei der IRC-Variante benötigt der Controller:

```text
Twitch Bot Account
        │
        ▼
OAuth Access Token
        │
        ▼
Twitch IRC
        │
        ▼
JOIN #channel
        │
        ▼
PRIVMSG empfangen
        │
        ▼
Command Parser
```

Der Twitch-IRC-Server ist:

```text
irc.chat.twitch.tv:6697
```

Alternativ kann IRC über WebSocket verwendet werden:

```text
wss://irc-ws.chat.twitch.tv:443
```

Twitch dokumentiert beide Verbindungsarten. citeturn1search0

---

# 5.6 Twitch-Bot-Account

Für die IRC-Variante wird ein Twitch-Account verwendet, über den der Controller den Chat liest.

Beispiel:

```text
Bot-Account:
streamcontroller
```

Dieser Account muss nicht der Streamer-Account sein.

Der Bot benötigt einen OAuth-Zugang mit mindestens:

```text
chat:read
```

Wenn der Bot selbst Antworten in den Chat schreiben soll:

```text
chat:edit
```

Twitch dokumentiert diese Scopes für IRC. citeturn1search5

---

# 5.7 Twitch IRC – Authentifizierung

Der IRC-Client verbindet sich und authentifiziert sich sinngemäß mit:

```text
PASS oauth:ACCESS_TOKEN
NICK botusername
```

Danach wird der Kanal betreten:

```text
JOIN #kanalname
```

Twitch verwendet für Chatnachrichten `PRIVMSG`. citeturn1search0

Der Controller muss außerdem auf `PING` reagieren:

```text
Twitch:
PING :tmi.twitch.tv

Controller:
PONG :tmi.twitch.tv
```

Ohne die Antwort kann Twitch die Verbindung schließen. citeturn1search0

---

# 5.8 Twitch IRC – Command Listener

Der Controller empfängt beispielsweise:

```text
:user!user@user.tmi.twitch.tv PRIVMSG #channel :!streamtwitch on
```

Der Controller zerlegt die Nachricht:

```text
Benutzer:
user

Kanal:
channel

Nachricht:
!streamtwitch on
```

Dann prüft er:

```text
Ist die Nachricht ein erlaubter Befehl?
        │
        ▼
Ist der Benutzer berechtigt?
        │
        ▼
Befehl ausführen
```

---

# 5.9 Berechtigungen

Nicht jeder Zuschauer darf:

```text
!streamtwitch on
```

ausführen.

Der Controller muss beispielsweise prüfen:

```text
Streamer
    ✓

Moderator
    ✓

Normaler Zuschauer
    ✗
```

Dazu können die Twitch-Chat-Metadaten ausgewertet werden.

Twitch kann mit der `twitch.tv/tags`-Capability zusätzliche Informationen zu Chatnachrichten liefern, darunter Benutzer- und Rolleninformationen. citeturn1search0

---

# 5.10 Twitch IRC – benötigte Capabilities

Für den Listener sollte mindestens sinnvoll konfiguriert werden:

```text
twitch.tv/commands
twitch.tv/tags
```

Beispiel:

```text
CAP REQ :twitch.tv/commands twitch.tv/tags
```

Dadurch können unter anderem die zusätzlichen Benutzerinformationen der Chatnachrichten ausgewertet werden. citeturn1search0

---

# 5.11 Chat-Befehle definieren

Die Befehle werden **im Controller** festgelegt.

Geplant:

```text
!streamtwitch on
!streamtwitch off

!streamyoutube on
!streamyoutube off

!streamstatus
```

Wichtig:

`!youtube` wird bewusst nicht verwendet, da dieser Name möglicherweise bereits als normaler Link-/Informationsbefehl genutzt wird.

---

# 5.12 Command-Verarbeitung

Vereinfacht:

```text
Chatnachricht
      │
      ▼
"!streamtwitch on"
      │
      ▼
Command Parser
      │
      ├── Command erkannt?
      │       │
      │       ▼
      │    Berechtigt?
      │       │
      │       ▼
      │    Aktion
      │
      └── Nein → ignorieren
```

Beispiel:

```text
!streamtwitch on
```

führt zu:

```text
Twitch Output = START
```

---

# 5.13 Statusmodell

Der Controller verwendet intern:

```text
INPUT
  ONLINE
  OFFLINE

TWITCH
  OFF
  STARTING
  LIVE
  STOPPING
  ERROR

YOUTUBE
  OFF
  STARTING
  LIVE
  STOPPING
  ERROR
```

Beispiel:

```text
Input:   ONLINE
Twitch:  LIVE
YouTube: OFF
```

---

# 5.14 `!streamstatus`

Bei:

```text
!streamstatus
```

soll der Controller beispielsweise antworten:

```text
Input: ONLINE | Twitch: LIVE | YouTube: OFF
```

Damit kann direkt im Chat kontrolliert werden, welcher Ausgang aktiv ist.

---

# 5.15 YouTube Chat

YouTube verwendet hierfür nicht denselben IRC-Mechanismus wie Twitch.

Die YouTube Live Streaming API stellt Live-Chat-Nachrichten über die `liveChatMessages`-Ressource bereit. Für neue Nachrichten gibt es auch `streamList`, eine serverseitige Verbindung mit geringer Latenz. citeturn0search1turn0search4

Die Architektur lautet daher:

```text
Controller
    │
    │ YouTube API
    ▼
YouTube Live Chat
    │
    ▼
Chatnachricht
    │
    ▼
Command Parser
```

Die `liveChatId` gehört zur jeweiligen Live-Übertragung und wird aus der Broadcast-Konfiguration ermittelt. citeturn0search1

---

# 5.16 YouTube Chat-Befehle

Der gleiche Command Parser kann verwendet werden:

```text
!streamtwitch on
!streamtwitch off

!streamyoutube on
!streamyoutube off

!streamstatus
```

Dadurch kann beispielsweise auch ein YouTube-Moderator einen Befehl ausführen.

Die Benutzerrolle aus YouTube muss dabei ebenfalls geprüft werden.

Die YouTube-API liefert entsprechende Autor-/Rolleninformationen über `authorDetails`. citeturn0search2

---

# 5.17 Twitch und YouTube gemeinsam behandeln

Der Controller sollte Chatnachrichten zunächst in ein gemeinsames internes Format umwandeln:

```text
source:
  twitch

user:
  username

role:
  moderator

message:
  !streamtwitch on
```

oder:

```text
source:
  youtube

user:
  username

role:
  moderator

message:
  !streamtwitch on
```

Danach wird derselbe Command Parser verwendet.

Vorteil:

```text
Twitch Chat ───┐
               ├──► Command Parser ──► Aktion
YouTube Chat ──┘
```

---

# 5.18 Twitch-Ausgabe

Der Controller soll Twitch unabhängig schalten können.

```text
MediaMTX
   │
   │ RTMPS
   ▼
Twitch
```

Status:

```text
OFF
STARTING
LIVE
STOPPING
ERROR
```

---

# 5.19 YouTube-Ausgabe

YouTube benötigt zusätzlich die Verwaltung des Live-Broadcasts.

Der Controller muss daher unterscheiden zwischen:

```text
YouTube Broadcast
```

und:

```text
YouTube Stream / Ingest
```

Die API übernimmt die Verwaltung des Broadcast-Lebenszyklus.

Die genaue API-Konfiguration wird in einem separaten Einrichtungsschritt durchgeführt, sobald die YouTube-Credentials erstellt wurden.

---

# 5.20 Secrets

Keine Zugangsdaten direkt in Python-Code speichern.

Vorgesehen:

```text
/opt/streaming/controller/.env
```

Beispiel:

```text
TWITCH_CLIENT_ID=...
TWITCH_CLIENT_SECRET=...
TWITCH_ACCESS_TOKEN=...
TWITCH_BOT_USERNAME=...
TWITCH_CHANNEL=...

YOUTUBE_CLIENT_ID=...
YOUTUBE_CLIENT_SECRET=...
YOUTUBE_REFRESH_TOKEN=...
```

Die echten Werte werden erst bei der Plattform-Einrichtung eingetragen.

Die Datei muss vor Zugriff durch andere Benutzer geschützt werden.

Beispielsweise:

```bash
chmod 600 /opt/streaming/controller/.env
```

---

# 5.21 Controller als Docker-Container

Geplante Datei:

```bash
nano /opt/streaming/controller/compose.yml
```

Die endgültige Compose-Datei wird erstellt, sobald die konkrete Controller-Implementierung festgelegt ist.

Ziel:

```text
controller
   │
   ├── Twitch API / Chat
   ├── YouTube API / Chat
   ├── MediaMTX-Steuerung
   └── Webinterface
```

Der Controller soll ebenfalls:

```text
restart: unless-stopped
```

verwenden.

---

# 5.22 Webinterface

Das Webinterface zeigt mindestens:

```text
INPUT
● ONLINE

Video:
1920x1080 @ 60 FPS

Bitrate:
6200 Kbit/s

TWITCH
● LIVE
[ AUS ]

YOUTUBE
○ OFF
[ AN ]

FALLBACK
● READY
```

Das Interface darf nicht ungeschützt im Internet stehen.

Es benötigt Authentifizierung.

---

# 5.23 Twitch und YouTube unabhängig schalten

Beispiel:

### 13:00

```text
Input:   ONLINE
Twitch:  OFF
YouTube: LIVE
```

### 13:45

```text
Input:   ONLINE
Twitch:  LIVE
YouTube: LIVE
```

### 14:00

```text
Input:   ONLINE
Twitch:  LIVE
YouTube: OFF
```

Der OBS-Eingang bleibt die ganze Zeit aktiv.

---

# 5.24 Fallback mit Plattformen

Wenn der Streamer ausfällt:

```text
OBS
 X
 │
 ▼
MediaMTX
 │
 ▼
fallback.mp4
 │
 ├──► Twitch
 └──► YouTube
```

Wenn nur Twitch aktiv ist:

```text
fallback.mp4
     │
     └──► Twitch
```

Wenn nur YouTube aktiv ist:

```text
fallback.mp4
     │
     └──► YouTube
```

Wenn beide aktiv sind:

```text
fallback.mp4
     ├──► Twitch
     └──► YouTube
```

---

# 6. Testplan

## Test 1 – Server

```text
SSH       ✓
UFW       ✓
Docker    ✓
Compose   ✓
```

---

## Test 2 – MediaMTX

```text
Container läuft
SRT-Port geöffnet
MediaMTX lauscht
```

---

## Test 3 – OBS → SRT

```text
OBS
 ↓
SRT
 ↓
MediaMTX
```

Prüfen:

- H264
- AAC
- 1920×1080
- 60 FPS

---

## Test 4 – Fallback

OBS stoppen:

```text
Live
 ↓
Fallback
```

OBS wieder starten:

```text
Fallback
 ↓
Live
```

---

## Test 5 – Twitch

```text
MediaMTX
 ↓
Twitch
```

Nur Twitch aktivieren.

---

## Test 6 – YouTube

```text
MediaMTX
 ↓
YouTube
```

Nur YouTube aktivieren.

---

## Test 7 – Beide Plattformen

```text
MediaMTX
 ├──► Twitch
 └──► YouTube
```

---

## Test 8 – Twitch Chat

Im Twitch-Chat:

```text
!streamstatus
```

Erwartung:

```text
Input: ONLINE | Twitch: ... | YouTube: ...
```

Danach:

```text
!streamtwitch on
```

und:

```text
!streamtwitch off
```

Nur berechtigte Benutzer dürfen die Befehle ausführen.

---

## Test 9 – YouTube Chat

Im YouTube-Livechat:

```text
!streamstatus
```

Danach:

```text
!streamyoutube on
```

und:

```text
!streamyoutube off
```

---

## Test 10 – Plattformen unabhängig schalten

Während YouTube läuft:

```text
!streamtwitch on
!streamtwitch off
```

YouTube darf nicht beeinflusst werden.

Während Twitch läuft:

```text
!streamyoutube on
!streamyoutube off
```

Twitch darf nicht beeinflusst werden.

---

## Test 11 – Unberechtigten Benutzer testen

Ein normaler Zuschauer versucht:

```text
!streamtwitch off
```

Erwartung:

```text
Keine Aktion
```

---

## Test 12 – Streamer-Ausfall

OBS stoppen.

Erwartung:

```text
Live
 ↓
Fallback
```

Die aktiven Plattformen bleiben aktiv.

---

## Test 13 – Streamer-Reconnect

OBS wieder starten.

Erwartung:

```text
Fallback
 ↓
Live
```

---

## Test 14 – Server-Neustart

```bash
sudo reboot
```

Nach Neustart:

```bash
docker ps
```

Prüfen:

- MediaMTX startet automatisch
- Controller startet automatisch
- Konfiguration bleibt erhalten
- Fallback bleibt verfügbar

---

## Test 15 – Ressourcen

Während eines echten Streams:

```bash
docker stats
```

Zusätzlich:

```bash
free -h
```

CPU und Netzwerk kontrollieren.

---

# 7. Zielzustand

Das fertige System:

```text
                           ┌────────────► Twitch
                           │
                           │
OBS / PC ─────SRT─────────►│
                           │
                           │ MediaMTX
                           │
IRL / Moblin ───SRTLA─────►│
                           │
                           └────────────► YouTube
                                    ▲
                                    │
                               Controller
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
               Twitch Chat                    YouTube Chat
                    │                               │
                    └───────────────┬───────────────┘
                                    │
                              Command Parser
                                    │
                              Webinterface
```

---

# 8. Spätere Erweiterungen

## 8.1 IRL / SRTLA

Später:

```text
Moblin
 │
 │ SRTLA
 ▼
Server
 │
 ▼
MediaMTX
```

Mögliche Erweiterungen:

- mehrere Mobilfunkverbindungen
- stabilerer IRL-Uplink
- SRTLA-Statistiken
- automatische Wiederverbindung

---

## 8.2 Monitoring

Später können überwacht werden:

- Input online/offline
- aktuelle Bitrate
- SRT-Latenz
- Paketverlust
- CPU
- RAM
- Netzwerk
- Twitch-Status
- YouTube-Status
- Fallback aktiv
- Controller aktiv

---

## 8.3 Benachrichtigungen

Mögliche Meldungen:

```text
Streamer offline
Fallback aktiviert
Streamer wieder online
Twitch offline
YouTube offline
Server neugestartet
Controller-Fehler
```

---

# 9. Wichtige Grundregel für die Installation

Die Installation wird **nicht komplett blind durchkopiert**.

Immer:

```text
Schritt ausführen
      ↓
Ausgabe prüfen
      ↓
Fehler beheben
      ↓
erst dann nächster Schritt
```

Insbesondere bei:

- SSH
- Firewall
- MediaMTX
- SRT
- OAuth
- Twitch
- YouTube

sollte jeweils zuerst kontrolliert werden, ob der aktuelle Schritt funktioniert.

---

# 10. Kurzüberblick der fertigen Komponenten

```text
┌─────────────────────────────────────────────┐
│                 Netcup VPS                  │
│                                             │
│  Ubuntu 24.04                               │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │ MediaMTX                              │  │
│  │                                       │  │
│  │ SRT Input                             │  │
│  │ live                                  │  │
│  │ fallback.mp4                          │  │
│  └───────────────────────────────────────┘  │
│                    │                        │
│                    ▼                        │
│  ┌───────────────────────────────────────┐  │
│  │ Controller                            │  │
│  │                                       │  │
│  │ Twitch                                │  │
│  │ YouTube                               │  │
│  │ Chat                                  │  │
│  │ Commands                              │  │
│  │ Permissions                           │  │
│  │ Webinterface                          │  │
│  └───────────────────────────────────────┘  │
│                                             │
└─────────────────────────────────────────────┘
```

Damit sind die Zuständigkeiten klar getrennt:

```text
MediaMTX
= Streaming

Controller
= Steuerung

Twitch / YouTube
= Plattformen

Chat
= Bedienung

Webinterface
= grafische Bedienung

SRTLA
= späterer IRL-Eingang
```
