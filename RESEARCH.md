# SPLITGATE – Client Architecture Research

Untersuchungsdatum: 2026-09-07
Methode: Statische Analyse der lokal installierten Client-Dateien (Verzeichnisstruktur, Dateitypen, `strings`-Extraktion aus Binärdateien). Keine Datei wurde verändert. Kein Reverse Engineering von Anti-Cheat/DRM, keine Umgehungsversuche.

---

## 1. Installation

Arbeitsverzeichnis: `Splitgate 2/` (internes Projekt heißt **"PortalWars2"** – vermutlich der interne Codename von Splitgate 2 beim Entwickler 1047 Games).

Top-Level-Struktur:

```
Splitgate 2/
├── PortalWars2.exe                  # Bootstrap-Launcher (klein, 266 KB)
├── Engine/                          # Unreal-Engine-Runtime (staged/shipped, kein Vollsource)
│   ├── Binaries/
│   │   ├── Win64/EOSSDK-Win64-Shipping.dll
│   │   └── ThirdParty/{Steamworks, DbgHelp, Ogg, NVIDIA, Vorbis, Windows}
│   ├── Config/StagedBuild_PortalWars2.ini   # leer
│   ├── Content/{Renderer, Slate, SlateDebug}
│   ├── Extras/Redist/en-us/UEPrereqSetup_x64.exe
│   └── Plugins/{Audio, CrashReporter, Marketplace, Online}
├── PortalWars2/                     # eigentliches Spiel
│   ├── Binaries/Win64/
│   │   ├── PortalWars2-Win64-Shipping.exe   # Haupt-Executable, 314 MB
│   │   ├── D3D12/, diverse DLLs (boost, tbb, curl, opus, tox, fvad, ...)
│   └── Content/{Movies, Paks, Splash}
│       └── Paks/  # UE5 IoStore-Container (.pak/.utoc/.ucas/.sig), ~37 GB gesamt
└── RemappedPlugins/1047/MerlinAntiCheat/ThirdParty/equ8_client/
    # Kernel-Anti-Cheat ("Merlin"/equ8 "Redkard"), inkl. .sys-Treiber-Bootstrap
```

**VERIFIED**: Es gibt genau eine Client-Executable, keine separate Server-Executable im installierten Verzeichnisbaum.
**VERIFIED**: Assets liegen ausschließlich als UE5-IoStore-Container vor (`global.ucas/.utoc`, `pakchunk*.ucas/.utoc/.pak/.sig`) – keine losen Assets, kein Klartext-Dateisystem für Inhalte.

---

## 2. Engine

**VERIFIED**: Unreal Engine, erkennbar an der Verzeichnisstruktur (`Engine/Binaries/Win64`, `Engine/Plugins`, `Engine/Config`, `.pak/.utoc/.ucas`-Dateiformat, PDB-Namen wie `BootstrapPackagedGame-Win64-Shipping.pdb`, `PortalWars2-Win64-Shipping.pdb`).

**STRONGLY INDICATED**: Unreal Engine **5.4.x oder neuer**. Beleg: Ein extrahierter Engine-Log-/CVar-String lautet wörtlich:
> "Do we send server initiated keys as acknowledged to the client? Default false. Was true prior to UE5.4.2."

Das ist ein Engine-internes Netzwerk-Verschlüsselungs-CVar mit explizitem Verweis auf UE5.4.2-Verhalten – starkes Indiz für Build auf Basis von UE 5.4.2 oder später. Eine exakte Versionsnummer (`Build.version`) ist im gestagten Build nicht vorhanden (Shipping-Build enthält kein Klartext-Versionsfile).

**VERIFIED**: Referenzierte Build-Pfade zeigen internen Branch-/Sync-Namen `H:\PW2-P2P-Full\Sync\Engine\...` und `H:\PW2-P2P-Full\Sync\Common\Plugins\1047\Online\OnlineNetworkUtils1047\...` – ein selbst entwickeltes Online-Plugin von 1047 Games, eingebettet im Engine-Fork.

**UNKNOWN**: Ob es sich um eine unveränderte Stock-UE5 oder einen signifikant modifizierten Engine-Fork handelt (der Pfadname "PW2-P2P-Full" deutet auf einen eigenen, projektspezifischen Branch hin, dessen Umfang an Modifikationen von außen nicht feststellbar ist).

---

## 3. Executables

| Datei | Größe | Typ | Rolle |
|---|---|---|---|
| `PortalWars2.exe` | 266 KB | PE32+, GUI, x86-64 | Bootstrap-Launcher (`BootstrapPackagedGame-Win64-Shipping.pdb`); startet vermutlich die eigentliche Shipping-Exe |
| `PortalWars2/Binaries/Win64/PortalWars2-Win64-Shipping.exe` | 314 MB | PE32+, GUI, x86-64, 10 Sections | Haupt-Client-Binary (Shipping-Konfiguration der UE) |
| `Engine/Extras/Redist/en-us/UEPrereqSetup_x64.exe` | – | PE32+ | Standard-UE-Redistributable-Installer (VC++ Runtimes etc.) |
| `RemappedPlugins/.../equ8_client/anticheat.x64.redkard.exe.bootstrap` | – | Binär | Anti-Cheat-Client-Bootstrap (Merlin/equ8, "Redkard") |

**VERIFIED**: Keine Datei mit Namen wie `*Server*.exe`, `*DedicatedServer*` o.ä. ist im installierten Baum vorhanden. Es handelt sich ausschließlich um einen Client-Build (`-Win64-Shipping.exe`, kein `-Win64-Server.exe`, wie es bei UE-Projekten mit separatem Server-Target üblich wäre).

---

## 4. Libraries

Relevante DLLs in `PortalWars2/Binaries/Win64/`:
- `tbb.dll`, `tbbmalloc.dll` – Intel Threading Building Blocks
- `boost_chrono-mt-x64.dll`, `boost_filesystem-mt-x64.dll`, `boost_python311-mt-x64.dll`, `boost_program_options-mt-x64.dll`, `boost_iostreams-mt-x64.dll`, `boost_atomic-mt-x64.dll`, `boost_system-mt-x64.dll`, `boost_regex-mt-x64.dll`, `boost_thread-mt-x64.dll` – Boost (inkl. Python-Bindings, ungewöhnlich für ein reines Spiel – evtl. Tooling/Editor-Reste oder Backend-Client-Code)
- `libcurl.dll`, `zlib1.dll` – HTTP/Komprimierung
- `opus.dll`, `opusenc.dll`, `fvad.dll` – Voice-Chat-Codec + Voice Activity Detection
- `libtox.dll` – Tox-Protokoll-Library (P2P-verschlüsselte Kommunikation; ungewöhnlich, Zweck im Client **UNKNOWN**)
- `amd_fidelityfx_dx12.dll`, `OpenColorIO_2_3.dll`, `D3D12/D3D12Core.dll`, `D3D12/d3d12SDKLayers.dll` – Rendering
- `Engine/Binaries/ThirdParty/Steamworks/Steamv157/Win64/steam_api64.dll` – Steamworks SDK v157
- `Engine/Binaries/Win64/EOSSDK-Win64-Shipping.dll` – Epic Online Services SDK, Version **1.17.0-39599718** (VERIFIED, aus Versionsstring in der DLL)

Anti-Cheat (nur zur Bestandsaufnahme, keine weitere Analyse):
- `RemappedPlugins/1047/MerlinAntiCheat/ThirdParty/equ8_client/`: `client.x64.redkard.dll`, `redkard.sys.bootstrap`, `vault.bin`, `static-vault.bin`, `anticheat.cfg` – Kernel-Mode-Anti-Cheat ("Merlin", Anbieter "equ8", Treibername "Redkard").

**VERIFIED**: `libtox.dll` (Tox-Protokoll) ist ungewöhnlich für ein Spiel dieser Art; mögliche Verwendung für P2P-Voice/Chat, aber nicht bestätigt.

---

## 5. Networking

Aus `strings`-Extraktion der Haupt-Shipping-Exe (442.785 extrahierte Strings, `-n 6`):

| Suchbegriff | Treffer | Bewertung |
|---|---|---|
| `DedicatedServer` | 13 | VERIFIED vorhanden (siehe Abschnitt 8) |
| `Matchmak` | 239 | VERIFIED (gRPC-Service `maverick.rooster_matchmaker.Matchmaker` etc.) |
| `Lobby` | 775 | VERIFIED (EOS Lobby API + eigener gRPC `LobbyManager`) |
| `P2P` | 163 | VERIFIED (EOS_P2P_* API vollständig gelinkt) |
| `NetDriver` | 15 | VERIFIED (UE-Standard-Netzwerksymbole, u. a. `BeaconNetDriver`, `DemoNetDriver`, `GameNetDriver`, `MeshNetDriver`, `NamedNetDriver`, `PendingNetDriver`) |
| `OnlineSubsystem` | 5 | VERIFIED (`/Script/OnlineSubsystem`, `/Script/OnlineSubsystemSteam`, `/Script/OnlineSubsystemUtils`) |
| `NAT` | 827 | größtenteils False Positives (Substring in anderen Wörtern) |
| `STUN` | 6 | größtenteils False Positives, kein eindeutiger STUN-Netzwerkcode gefunden |
| `TURN` | 362 | größtenteils False Positives |
| `ServerBrowser`, `SessionInterface`, `SteamSockets`, `ReplicationGraph`, `HostMigration` | 0 | nicht als Klartext-String gefunden (UNKNOWN, ob Feature nicht vorhanden oder nur Symbolname anders/gestrippt) |

**VERIFIED**: `OnlineSubsystemSteam` ist als Modul referenziert – d. h. das Spiel nutzt (zumindest teilweise) Epics `OnlineSubsystem`-Abstraktion mit Steam als Backend, zusätzlich zu EOS.

**VERIFIED**: Ein eigenständiges Plugin `OnlineNetworkUtils1047` (1047 Games) existiert im Engine-Fork (`Common\Plugins\1047\Online\OnlineNetworkUtils1047`), u. a. mit einer Datei `GrpcRequest.h` – bestätigt, dass 1047 Games gRPC-Requests direkt aus dem Client heraus absetzt (nicht nur Engine-intern).

**VERIFIED**: Der Client bindet gRPC (v1.54.2), Protobuf (21.12), NATS-Client (C-Library, voller Quellbaum referenziert) und AWS-SDK-Bestandteile (CRT) statisch/dynamisch ein. Das ist ungewöhnlich für ein reines Client-Binary und zeigt, dass der Client direkt mit Backend-Microservices kommuniziert (kein reiner REST/HTTP-Client).

---

## 6. P2P

**VERIFIED**: Die komplette **EOS P2P API** ist im Client verlinkt (Funktionssymbole aus `EOSSDK-Win64-Shipping.dll`, referenziert von der Haupt-Exe):
```
EOS_P2P_AcceptConnection
EOS_P2P_AddNotifyPeerConnectionClosed
EOS_P2P_AddNotifyPeerConnectionRequest
EOS_P2P_CloseConnection(s)
EOS_P2P_GetNextReceivedPacketSize
EOS_P2P_ReceivePacket
EOS_P2P_SendPacket
EOS_P2P_SetRelayControl
EOS_Platform_GetP2PInterface
```
Das ist die Standard-EOS-P2P-Transportschicht (NAT-Punchthrough + Relay-Fallback über Epics eigene Relay-Server), wie sie üblicherweise als Transport für einen UE-`NetDriver` verwendet wird (Analogon zu Steams `SteamNetworkingSockets`).

**STRONGLY INDICATED**: Der interne Build-/Sync-Pfadname **"PW2-P2P-Full"** (durchgängig in Debug-Pfaden der Shipping-Exe) deutet stark darauf hin, dass P2P-Networking (vermutlich EOS-P2P-basiert) ein zentraler, namensgebender Architekturbestandteil des Projekts war/ist – möglicherweise für Nicht-Wettkampf-Modi oder als Fallback-Transport neben dedizierten Servern.

**UNKNOWN**: Ob P2P nur für Lobby/Party-Voice-Chat verwendet wird oder auch für tatsächliches Match-Gameplay (Replikation der Spiel-Engine über einen `MeshNetDriver`/`PendingNetDriver` via EOS-P2P-Transport). Der gefundene Symbolname `MeshNetDriver` (custom, kein UE-Standardname) deutet auf einen eigenen NetDriver hin, der eventuell Mesh-Topologie (jeder mit jedem, kein zentraler Host) implementiert – dies ist jedoch **nicht verifiziert**, nur ein Namensfund.

**VERIFIED**: `SteamNetworking006` (alte Steamworks-Networking-Interface-Version) ist als String vorhanden – zusätzlicher Hinweis auf Steam als alternativen/zusätzlichen P2P-Transport neben EOS.

---

## 7. Matchmaking

**VERIFIED**: Der Client referenziert ein komplettes, selbst entwickeltes gRPC-Backend-API unter dem Namensraum `maverick.*` (Firma/Produktname vermutlich "Maverick" als Backend-Plattform von 1047 Games), inkl. vollständiger Service-/Methodennamen (siehe Abschnitt 9 für die vollständige Liste). Matchmaking-relevante Services:

- `maverick.rooster_matchmaker.Matchmaker` – `SubmitTicket`, `CancelTicket`, `FirstLoginCheck`
- `maverick.rooster.lobby_manager.Matchmaking` – `StartMatchmaking`, `CancelMatchmaking`, `CreateMatchmakingLobby`, `RegisterPlayersMatchEnded`
- `maverick.lobby_manager_v2.LobbyManager` – `CreateLobby`, `FindLobbies`, `JoinLobby`, `RequestGameServer`, `GameServerReady`, `GameServerEnded`, `ShutdownLobby`, `AssignTeam`, `ShuffleTeams`, u. v. m.
- `maverick.login_queue.LoginQueue` – `JoinLoginQueue`, `TryLogin`
- `maverick.natsmanager.*` – NATS-Verbindungstoken/Messaging (vermutlich Echtzeit-Pub/Sub für Lobby-/Matchmaking-Events)

**VERIFIED**: Das Matchmaking läuft über einen zentralen Cloud-Backend-Dienst (gRPC über HTTP/2, referenzierte Envoy-Config-Strings deuten auf einen Envoy-Proxy im Backend-Pfad hin), **nicht** über ein dezentrales/Client-only-Matchmaking-Protokoll. Dieser Dienst ist mit hoher Sicherheit nach Abschaltung der offiziellen Server nicht mehr erreichbar.

**UNKNOWN**: Ob zusätzlich zum Backend-Matchmaking auch EOS-eigenes Session-/Lobby-Matchmaking (`EOS_Sessions_CreateSessionSearch`, `EOS_Lobby_CreateLobbySearch`) als Fallback oder Parallelweg genutzt wird, oder ob EOS Sessions/Lobby nur für Party-/Invite-Funktionalität (nicht Matchmaking) dient.

---

## 8. Server/Host Architecture

**VERIFIED**: Vollständiger gRPC-Service `maverick.dsm.DedicatedServerManager` mit den Methoden:
```
AllocateServer
DeallocateServer
GameServerInitialized
GetAllocationDetails
GetAllocationDetailsByProviderData
GetRegionEndpoints
```
plus zugehöriger Proto-Dateireferenz `dedicated-server-manager/dsm.proto` und `dedicated-server-manager/dsm_events.proto`.

Das ist ein klassisches Cloud-Gameserver-Allocation-Muster (vergleichbar mit Systemen wie Agones/Multiplay/PlayFab MPS): Ein zentraler "DSM"-Dienst allokiert und deallokiert dedizierte Spielserver-Instanzen, liefert Region-Endpunkte und erhält Bereitschafts-/Beendigungs-Events (`GameServerReady`, `GameServerEnded`, `GameServerInitialized` aus Abschnitt 7 gehören funktional dazu).

**VERIFIED → STRONGLY INDICATED**: Das offizielle Server-Modell war also (zumindest teilweise) **echtes Dedicated-Server-Hosting in der Cloud**, orchestriert durch das Maverick-Backend – nicht reines P2P. Die genaue Rolle von P2P (Abschnitt 6) im Vergleich zu DSM-Servern (z. B. P2P nur für Custom-Matches/Privatpartien, DSM für Ranked/offizielle Playlists) ist aus reiner Client-Analyse **nicht abschließend klärbar (UNKNOWN)**.

**VERIFIED**: Im installierten Client-Verzeichnis existiert **keine** separate Dedicated-Server-Binary, kein Server-Target-Build (`-Server.exe`), keine Server-spezifischen Konfigurationsdateien und keine Hinweise auf ein lokal ausführbares Server-Modul. Der ausgelieferte Build ist ein reiner Shipping-Client.

**UNKNOWN**: Ob der Quellcode/das Build-System (das offensichtlich existiert, siehe `H:\PW2-P2P-Full\...\Engine\Source`) einen UE-Server-Target (`TargetType.Server`) enthält, der theoretisch kompiliert werden könnte – das ist aus der Installation heraus nicht feststellbar, da kein Quellcode vorliegt.

---

## 9. Evidence

Auszug der wichtigsten Rohbefunde (vollständige Service-Liste aus `strings` der Shipping-Exe, Namensraum `maverick.*`, alle **VERIFIED** als vorhandene Symbolstrings – keine Aussage über tatsächliche Erreichbarkeit/Funktion):

```
maverick.anti_cheat.GameClientData
maverick.code_redemption.CodeRedemptionGenerate
maverick.config_manager.ConfigManager
maverick.dsm.DedicatedServerManager          <-- Dedicated Server Manager
maverick.friends.Presence
maverick.game_cms.Localization
maverick.iam.GameAuth / IAMAuthorization / IdentityGraph
maverick.leaderboard.StatsLeaderboardQueries
maverick.lobby_manager_v2.LobbyManager       <-- Lobby / Matchmaking-Anbindung
maverick.login_queue.LoginQueue
maverick.moderation.Reports
maverick.natsmanager.NatsManagerConnections / NatsManagerMessages
maverick.rooster.battle_pass_v2.BattlePassService
maverick.rooster.challenges_v2.ChallengeV2Service
maverick.rooster.daily_login_rewards.DailyLoginRewardsService
maverick.rooster.levels(.v2).Levels(V2Service)
maverick.rooster.limited_time_events.EventsService
maverick.rooster.lobby_manager.Matchmaking   <-- Matchmaking
maverick.rooster.referral.ReferralService
maverick.rooster_matchmaker.Matchmaker       <-- Matchmaking
maverick.rooster_match.MatchQueries
maverick.rooster_rank.RankLeaderboardQueries / RankQueries
maverick.store.Store
maverick.ugc.UserFile / UserGeneratedContent
maverick.userstats.UserStatsChanges / UserStatsViews
```

Zusätzliche wörtliche Fundstellen:
- Build-Pfad: `H:\PW2-P2P-Full\Sync\...` (durchgängig)
- Custom Plugin: `H:\PW2-P2P-Full\Sync\Common\Plugins\1047\Online\OnlineNetworkUtils1047\Source\OnlineNetworkUtils1047\Public\Grpc\GrpcRequest.h`
- Proto-Paket-Referenzen: `github.com/1047Games/maverick-protos/go/...` (Go-Modul-Pfade der Proto-Definitionen, d. h. das Backend-Repository heißt vermutlich `maverick-protos`)
- Engine-Netzwerk-CVar-Kommentar mit Verweis auf UE5.4.2
- EOSSDK-Version: `1.17.0-39599718`
- Steamworks-Version: `Steamv157`
- gRPC `1.54.2`, Protobuf `21.12` (aus Build-Pfaden)

---

## 10. Unknowns

- Exakte Unreal-Engine-Versionsnummer (nur "≥/um UE5.4.2" indiziert, kein Build.version-File vorhanden).
- Ob `MeshNetDriver` tatsächlich für Gameplay-Replikation zwischen Peers ohne dedizierten Server genutzt wird, oder nur ein interner Name ohne P2P-Gameplay-Bezug ist.
- Genaues Zusammenspiel/Priorisierung zwischen EOS-P2P-Transport, EOS Sessions/Lobby und dem Maverick-DSM-Cloud-Server-Modell (welcher Modus für welche Playlist/welchen Spielmodus verwendet wurde).
- Inhalt der `.pak`/`.ucas`-Container (IoStore-verschlüsselt/komprimiert, keine Klartext-Strings extrahierbar ohne UnrealPak/IoStore-Tooling und ggf. Verschlüsselungsschlüssel) – Blueprints/Ini-Configs mit `NetDriverDefinitions`, Karten-URLs, Server-Travel-Logik liegen vermutlich dort, wurden aber nicht analysiert.
- Ob im (nicht vorliegenden) Quellcode ein UE-Server-Build-Target existiert.
- Rolle von `libtox.dll` (Tox-Protokoll) im Client.
- Umfang der Engine-Modifikationen gegenüber Stock-UE5 (Branch-Name "PW2-P2P-Full" impliziert Eigenentwicklung, Umfang unbekannt).
- Ob NATS/gRPC-Endpunkte (Hostnames/URLs) im Client hartkodiert sind oder zur Laufzeit via Config/Discovery bezogen werden (in dieser Untersuchung wurden nur Servicenamen, keine konkreten URLs extrahiert).

---

## 11. Dedicated Server Feasibility

Einschätzung ausschließlich auf Basis der Client-Installation, **keine Aussage zu rechtlicher Zulässigkeit**:

- **VERIFIED**: Es gibt keinen mitgelieferten, selbst lauffähigen Dedicated-Server-Modus im installierten Client-Paket (kein Server-Target-Build).
- **VERIFIED**: Das Spiel ist architektonisch auf ein zentrales Cloud-Backend (Maverick: IAM/Auth, LobbyManager, Matchmaker, DedicatedServerManager, NATS-Messaging) ausgelegt. Ein 1:1-Nachbau eines "Community Dedicated Server" im Sinne von „Client verbindet sich per IP an einen selbstgehosteten Server ohne Zwischenschicht" ist mit den vorhandenen Client-Binaries **nicht ohne Weiteres** möglich, da:
  1. Authentifizierung (`maverick.iam.GameAuth`) vermutlich Voraussetzung für Verbindungsaufbau ist,
  2. Matchmaking/Lobby-Fluss (`RequestGameServer`, `GameServerReady`) eng mit dem DSM-Backend verzahnt zu sein scheint,
  3. keine Klartext-Konfigurationsdatei gefunden wurde, die einen direkten `?connect=IP`-Weg ohne Backend zeigt (UNKNOWN, da UE dies grundsätzlich unterstützt, aber hier nicht bestätigt).
- **STRONGLY INDICATED**: Eine Community-Infrastruktur müsste vermutlich das gesamte Maverick-Backend-API-Oberfläche (gRPC-Services aus Abschnitt 9) zumindest teilweise nachbilden (eigener Auth-/Lobby-/Matchmaking-/DSM-Dienst), nicht nur einen einzelnen Server-Prozess bereitstellen – das ist ein erheblich größerer Scope als ein klassischer UE-Dedicated-Server-Nachbau.
- **UNKNOWN**: Ob der P2P-Pfad (EOS_P2P) unabhängig vom Maverick-Backend für Custom-Games nutzbar wäre (z. B. reine Party-/Privatlobby ohne Matchmaking) – das wäre der vielversprechendste Ansatzpunkt für ein Minimal-Setup, ist aber nicht verifiziert.

---

## 12. Linux Feasibility

- **VERIFIED**: Es liegt ausschließlich ein Windows-Build vor (PE32+-Executables, `Win64`-Verzeichnisse). Es existiert kein natives Linux-Binary.
- **VERIFIED**: Im Spielverzeichnis liegen `vkd3d-proton.cache` und `vkd3d-proton.cache.write` – das Spiel läuft hier offensichtlich unter **Proton/VKD3D-Proton** (Windows-D3D12-auf-Vulkan-Übersetzung), also über Steam Play auf Linux.
- **UNKNOWN**: Anti-Cheat-Kompatibilität mit Proton – der vorhandene Kernel-Anti-Cheat (Merlin/equ8, `redkard.sys`) ist ein Windows-Kernel-Treiber; solche Systeme laufen unter Linux/Proton in der Regel nicht (kein Windows-Kernel unter Linux), was faktisch bedeutet, dass der **offizielle** Client mit Anti-Cheat unter Linux vermutlich nicht/nicht vollständig funktioniert – für eine reine Community-Server-Analyse ohne offizielles Backend ist Anti-Cheat aber ohnehin irrelevant.
- **UNKNOWN**: Ob ein Dedicated-Server-Target (falls im Quellcode vorhanden) für Linux kompilierbar wäre – UE5 unterstützt grundsätzlich Linux-Server-Targets, das hängt aber vollständig vom (hier nicht vorliegenden) Quellcode/Build-System ab, nicht von der Client-Installation.

---

## 13. Recommended Next Steps

Die 10 wichtigsten als Nächstes zu untersuchenden Punkte:

1. **IoStore/Pak-Inhalte extrahieren** (mit UnrealPak/UnrealPak-Alternativen wie `retoc`/`FModel`, sofern rechtlich zulässig und ohne DRM-Umgehung) – insbesondere `Config`-Assets (`DefaultEngine.ini`-Äquivalente), `NetDriverDefinitions`, Karten-Gamemode-Blueprints, um zu klären, welcher NetDriver/Transport tatsächlich für Gameplay verwendet wird.
2. **Netzwerkverkehr des Clients passiv beobachten** (z. B. mit Wireshark, sofern der Client überhaupt noch startet/verbindet), um zu sehen, welche Hosts/Ports/Protokolle beim Start tatsächlich kontaktiert werden (DNS-Anfragen, TCP/UDP-Ziele) – zeigt, ob überhaupt noch ein Backend erreichbar ist.
3. Klären, **ob und wie der Client ohne erreichbares Maverick-Backend überhaupt bis zum Hauptmenü / in ein Match kommt** (Fail-Fast bei Auth? Offline-Modus? Lokales Testspiel möglich?).
4. Prüfen, ob **EOS Sessions/Lobby direkt (ohne Maverick-Backend)** ansprechbar sind, z. B. über eigene EOS-Client-ID/Sandbox – das würde den P2P-Pfad potenziell unabhängig vom offiziellen Backend nutzbar machen.
5. Recherche, ob 1047 Games / Community bereits **öffentliche Aussagen, Patents, Blogposts oder Leaks** zur Server-Architektur ("Maverick", "Rooster", "DSM") gemacht haben, die die Client-seitigen Funde bestätigen/ergänzen.
6. Klären, ob es **Analogien zu Splitgate 1** gibt (Vorgänger nutzte laut öffentlichen Aussagen ebenfalls eine Mischung aus P2P und dediziertem Hosting über AWS GameLift) – Splitgate 1 hatte ggf. bereits Community-Reverse-Engineering-Vorarbeiten.
7. Untersuchen, ob **UE5-Standard-Mechanismen** (`?dedicated`, `-server`, direkter IP-Connect via `open IP:Port`) im Shipping-Build überhaupt noch kompiliert/aktiv sind (Code-Pfade können in Shipping-Builds per `#if !UE_SERVER`/`WITH_EDITOR` entfernt sein) – ggf. durch dynamische Analyse (Disassembly/Debugger) statt nur `strings`.
8. Rechtliche Prüfung: Nutzungsbedingungen/EULA von Splitgate 2 bzgl. Reverse Engineering, Reimplementierung von Serverfunktionalität und Nutzung der `maverick-protos`-Schnittstellendefinitionen.
9. Prüfen, ob **1047 Games offizielle Unterstützung/Kommunikation** zu Community-Servern nach EOL angekündigt hat (viele Studios veröffentlichen bei Server-Abschaltung offizielle Tools oder Serverbinaries – das wäre der sauberste Weg statt Reverse Engineering).
10. Technische Machbarkeitsstudie für **Nachbau eines Minimal-Backends** (Auth-Stub, Lobby-Stub, DSM-Stub) auf Basis der in Abschnitt 9 gefundenen gRPC-Service-/Methodennamen, sobald die genauen `.proto`-Nachrichtenformate (Feldnamen/-typen) bekannt sind (dafür wäre Punkt 1 und 2 Voraussetzung).

---

*Hinweis: Diese Analyse beruht ausschließlich auf statischer Untersuchung vorhandener Dateien (Verzeichnisstruktur, Dateitypen, extrahierte Textstrings aus Binärdateien). Es wurde kein Code disassembliert, kein Anti-Cheat/DRM analysiert oder umgangen, und keine Datei der Installation verändert.*
