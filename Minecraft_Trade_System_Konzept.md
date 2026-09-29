# Minecraft `/trade` System – Vollständiges Entwicklungs- und Technikkonzept

**Dokumenttyp:** Technische Spezifikation / Entwicklungsbriefing  
**System:** Minecraft `/trade`  
**Zielplattform:** Minecraft 1.21.x / Paper- bzw. kompatible Server-Software  
**Netzwerk:** Proxy-basiertes Multi-Server-Netzwerk  
**Proxy:** Velocity  
**Persistenz:** SQL-Datenbank  
**Optionaler Shared Cache / Messaging:** Redis  
**Hauptziel:** Sicheres, serverübergreifendes Spieler-Handelssystem mit GUI, Item- und Geldhandel, Schutz vor Duplizierung und sauberem Recovery bei Disconnects/Serverwechseln.

---

# 1. Ziel des Systems

Das `/trade`-System ermöglicht zwei Spielern, Items und optional eine Geldsumme sicher miteinander zu tauschen.

Ein Trade darf erst abgeschlossen werden, wenn:

1. beide Spieler dieselben Trade-Inhalte bestätigt haben,
2. nach einer Änderungsphase beide Spieler erneut bestätigen,
3. beide Spieler den finalen Abschluss bestätigen,
4. das System serverseitig atomar feststellt, dass sämtliche Gegenstände und Geldwerte vorhanden sind,
5. die Transaktion erfolgreich durchgeführt wurde.

Das System muss verhindern:

- Item-Duplikation
- Money-Duplikation
- Verlust von Items
- Verlust von Geld
- Trades mit Offline-Spielern
- Manipulation durch Inventar-Events
- Manipulation durch Serverwechsel
- Manipulation durch Plugin-/Server-Neustarts
- doppelte Transaktionen
- Race Conditions
- ungültige Inventarbewegungen
- Trade-Abschluss während eines kritischen Disconnects

Das System soll sich für Spieler möglichst einfach anfühlen, intern jedoch transaktional und streng abgesichert arbeiten.

---

# 2. Grundarchitektur

Das System besteht aus mehreren Ebenen:

```text
                         ┌─────────────────────┐
                         │       Velocity      │
                         │       Proxy         │
                         └──────────┬──────────┘
                                    │
                   Proxy Messaging / Redis PubSub
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
     ┌──────▼──────┐        ┌──────▼──────┐        ┌──────▼──────┐
     │   Server 1  │        │   Server 2  │        │   Server N  │
     │   Citybuild │        │   Farmwelt   │        │   Lobby/... │
     └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
            │                       │                       │
            └───────────────────────┼───────────────────────┘
                                    │
                            ┌───────▼───────┐
                            │   SQL DB      │
                            │ Trades/Logs   │
                            └───────────────┘
```

## 2.1 Verantwortlichkeiten

### Backend-Plugin

Das Trade-Plugin auf jedem Minecraft-Server ist verantwortlich für:

- GUI
- Inventar-Events
- Spieler-Interaktion
- Trade-Anfrage
- lokale Item-Verwaltung
- lokale Validierung
- lokale Spielerinformationen
- Öffnen/Schließen des Trade-GUIs
- Sound/Animation
- Kommunikation mit dem zentralen Trade-Service
- Recovery nach Disconnect

### Velocity-Proxy

Der Proxy ist verantwortlich für:

- serverübergreifende Erreichbarkeit
- Weiterleitung von Trade-Nachrichten
- Ermittlung des Servers eines Spielers
- Cross-Server-Trade-Anfragen
- Synchronisation von Spielerstatus
- Routing der Trade-Kommunikation

Der Proxy soll **nicht** die Minecraft-Items selbst verwalten.

### SQL-Datenbank

Die Datenbank speichert:

- Trade-Transaktionen
- Trade-Status
- Trade-Teilnehmer
- Trade-Logs
- Recovery-Daten
- optionale Statistiken
- eindeutige Trade-IDs

### Redis – empfohlen

Redis ist für Echtzeitkommunikation sinnvoll:

- Pub/Sub
- Locks
- Serverübergreifende Events
- Trade-State-Synchronisation
- schnelle Online-State-Abfragen

SQL bleibt die dauerhafte Datenquelle.

---

# 3. Voraussetzungen

Benötigt:

- Java 21+
- Paper oder kompatibler Server
- Velocity
- LuckPerms oder anderes Permission-System
- Economy-System
- SQL-Datenbank
- optional Redis

Das Plugin sollte möglichst wenig von konkreten Economy-Plugins abhängig sein.

Dafür wird eine Economy-Abstraktion verwendet:

```text
EconomyProvider
 ├── VaultProvider
 ├── PlayerPointsProvider (optional)
 └── CustomEconomyProvider
```

Das Trade-System selbst kennt nur:

```text
getBalance(player)
withdraw(player, amount)
deposit(player, amount)
canAfford(player, amount)
```

---

# 4. Trade-Grundprinzip

Ein Trade besteht aus:

```text
TradeRequest
      ↓
PENDING
      ↓
ACCEPTED
      ↓
OPEN
      ↓
READY
      ↓
CONFIRMATION
      ↓
COMPLETED
```

Mögliche Abbruchzustände:

```text
CANCELLED
EXPIRED
DISCONNECTED
SERVER_SWITCH
ERROR
RECOVERY
```

---

# 5. Trade-ID

Jeder Trade erhält eine eindeutige ID.

Beispiel:

```text
TRD-20260929-7F42A9C1
```

Intern besser UUID:

```text
550e8400-e29b-41d4-a716-446655440000
```

Diese UUID wird für:

- Datenbank
- Logs
- Redis
- Debugging
- Recovery

verwendet.

---

# 6. Trade-Anfrage

Spieler A:

```text
/trade SpielerB
```

Spieler B erhält:

```text
╔══════════════════════════════════════╗
║          TRADE-ANFRAGE               ║
║                                      ║
║  SpielerA möchte mit dir handeln.   ║
║                                      ║
║       [ ANNEHMEN ] [ ABLEHNEN ]     ║
╚══════════════════════════════════════╝
```

Optional zusätzlich Chat:

```text
SpielerA möchte mit dir handeln.
Klicke auf [ANNEHMEN] oder [ABLEHNEN].
Die Anfrage läuft in 30 Sekunden ab.
```

---

# 7. Trade-Anfrage-Regeln

Standardwerte:

```yaml
request-expire-seconds: 30
request-cooldown-seconds: 3
max-active-trades-per-player: 1
```

Ein Spieler darf standardmäßig nur:

- eine aktive Trade-Session
- eine eingehende Anfrage
- eine ausgehende Anfrage

haben.

---

# 8. Commands

## Spielercommands

### `/trade <Spieler>`

Sendet eine Trade-Anfrage.

Permission:

```text
trade.use
```

---

### `/trade accept`

Nimmt die aktuelle Anfrage an.

Permission:

```text
trade.use
```

---

### `/trade deny`

Lehnt die aktuelle Anfrage ab.

Permission:

```text
trade.use
```

---

### `/trade cancel`

Bricht den aktuellen Trade ab.

Permission:

```text
trade.use
```

---

### `/trade toggle`

Aktiviert/deaktiviert Trade-Anfragen.

Permission:

```text
trade.toggle
```

---

### `/trade status`

Zeigt den aktuellen Trade-Status.

Permission:

```text
trade.status
```

---

# 9. Admin-Commands

### `/tradeadmin reload`

Lädt Konfiguration neu.

Permission:

```text
trade.admin.reload
```

---

### `/tradeadmin info <Trade-ID>`

Zeigt Trade-Daten.

Permission:

```text
trade.admin.info
```

---

### `/tradeadmin cancel <Trade-ID>`

Bricht einen Trade administrativ ab.

Permission:

```text
trade.admin.cancel
```

---

### `/tradeadmin recover <Trade-ID>`

Startet eine Recovery-Prüfung.

Permission:

```text
trade.admin.recover
```

---

### `/tradeadmin player <Spieler>`

Zeigt aktive Trades und Trade-Statistiken.

Permission:

```text
trade.admin.player
```

---

### `/tradeadmin debug`

Aktiviert Debug-Modus.

Permission:

```text
trade.admin.debug
```

---

# 10. Permissions

```text
trade.use
trade.toggle
trade.status

trade.admin
trade.admin.reload
trade.admin.info
trade.admin.cancel
trade.admin.recover
trade.admin.player
trade.admin.debug

trade.bypass.cooldown
trade.bypass.distance
trade.bypass.disabled
trade.bypass.limit

trade.economy
trade.items
trade.crossserver
```

Default:

```text
trade.use: true
trade.toggle: true
trade.status: true
```

Admin:

```text
trade.admin.*: false
```

---

# 11. Haupt-GUI

Empfohlen:

```text
6 Reihen × 9 Slots = 54 Slots
```

GUI:

```text
┌─────────────────────────────────────────────┐
│             TRADE MIT SpielerB              │
├─────────────────────────────────────────────┤
│                                             │
│   SPIELER A              SPIELER B          │
│                                             │
│   [00][01][02][03]    [05][06][07][08]     │
│   [09][10][11][12]    [14][15][16][17]     │
│   [18][19][20][21]    [23][24][25][26]     │
│                                             │
│       GELD                 GELD             │
│      [27]                 [35]              │
│                                             │
│                                             │
│   [ READY ]             [ READY ]           │
│                                             │
│          [ FINAL BESTÄTIGEN ]               │
└─────────────────────────────────────────────┘
```

---

# 12. Exakte Slotbelegung

Minecraft-Slotnummern:

```text
0  1  2  3  4  5  6  7  8
9 10 11 12 13 14 15 16 17
18 19 20 21 22 23 24 25 26
27 28 29 30 31 32 33 34 35
36 37 38 39 40 41 42 43 44
45 46 47 48 49 50 51 52 53
```

## Spieler A

Item-Slots:

```text
0
1
2
3
9
10
11
12
18
19
20
21
```

Maximal 12 Item-Slots.

## Spieler B

Item-Slots:

```text
5
6
7
8
14
15
16
17
23
24
25
26
```

Maximal 12 Item-Slots.

## Geld

Spieler A:

```text
27
```

Spieler B:

```text
35
```

## Ready

Spieler A:

```text
36
```

Spieler B:

```text
44
```

## Final Confirm

```text
49
```

## Cancel

```text
45
```

## Status

```text
4
13
22
31
40
```

---

# 13. Design

Das GUI soll modern und klar aussehen.

Empfohlenes Farbschema:

```text
Dunkelgrau / Schwarz
Grün = bereit
Rot = nicht bereit / abbrechen
Gold = Geld
Weiß = Items
Gelb = Warnungen
```

Keine unnötigen Glasflächen über den Item-Slots.

Item-Slots müssen optisch eindeutig vom Interface getrennt sein.

---

# 14. Item-Slots

Spieler können Items per:

- Linksklick
- Shift-Klick
- Drag
- Doppelklick
- Hotbar-Swap
- Number-Key
- Drop
- Pickup

nicht außerhalb der vorgesehenen Trade-Slots bewegen.

Das Plugin muss sämtliche Inventory-Events absichern.

---

# 15. Verbotene Items

Konfigurierbar:

```yaml
blocked-items:
  - BEDROCK
  - COMMAND_BLOCK
  - CHAIN_COMMAND_BLOCK
  - REPEATING_COMMAND_BLOCK
```

Zusätzlich sollte eine API für andere Plugins existieren:

```java
TradeItemValidator
```

Damit können andere Plugins Items blockieren.

---

# 16. NBT und Item-Daten

Items müssen vollständig erhalten bleiben.

Dazu gehören:

- Material
- Anzahl
- ItemMeta
- Name
- Lore
- Enchantments
- Attribute
- CustomModelData
- Potion-Daten
- Banner-Daten
- Firework-Daten
- Trim-Daten
- Komponenten
- PersistentDataContainer
- Plugin-spezifische Daten

Es darf niemals nur Material + Amount gespeichert werden.

---

# 17. Money-System

Geld kann über einen Slot gesetzt werden.

Slot:

```text
27 / 35
```

Beim Klick öffnet sich ein Eingabefenster bzw. Chat-Eingabe.

Beispiel:

```text
Wie viel Geld möchtest du anbieten?

Aktuelles Guthaben: $250,000

Schreibe die Summe in den Chat.
Schreibe "cancel" zum Abbrechen.
```

Beispiel:

```text
$50,000
```

Anzeige:

```text
Geldangebot
$50,000
```

---

# 18. Money-Validierung

Beim Setzen:

```text
amount > 0
amount <= balance
```

Beim finalen Abschluss muss das Guthaben erneut geprüft werden.

Wichtig:

Die Balance darf nicht nur beim Öffnen des GUIs geprüft werden.

Beispiel:

```text
Trade geöffnet
Spieler hat $100.000

Spieler gibt außerhalb des Trades $80.000 aus

Trade möchte $100.000 übertragen

→ Trade darf NICHT abgeschlossen werden.
```

---

# 19. Ready-System

Wenn Spieler A auf READY klickt:

```text
readyA = true
```

Wenn Spieler B anschließend ein Item verändert:

```text
readyA = false
readyB = false
```

Jede relevante Änderung setzt beide Spieler zurück.

Das betrifft:

- Items
- Money
- relevante Trade-Einstellungen

---

# 20. Bestätigungsphase

Wenn beide READY sind:

```text
READY A = true
READY B = true
```

beginnt eine kurze Sicherheitsphase.

Beispiel:

```text
3
2
1
```

Danach:

```text
FINAL BESTÄTIGEN
```

wird aktiviert.

Jede Änderung setzt den Zustand wieder zurück.

---

# 21. Warum die zusätzliche Bestätigung?

Sie verhindert, dass ein Spieler:

1. READY drückt
2. der andere sofort bestätigt
3. ein Item im letzten Moment verändert wird

Die finale Bestätigung muss auf einem unveränderten Snapshot basieren.

---

# 22. Trade Snapshot

Vor dem finalen Abschluss wird ein unveränderlicher Snapshot erstellt:

```text
TradeSnapshot {
    tradeId
    playerA
    playerB
    itemsA
    itemsB
    moneyA
    moneyB
    timestamp
}
```

Danach werden Hashes gebildet.

Beispiel:

```text
snapshotHash = SHA-256(...)
```

Beim Abschluss:

```text
aktueller Hash == Snapshot Hash
```

Nur dann darf abgeschlossen werden.

---

# 23. Trade-Abschluss

Der Abschluss muss atomar erfolgen.

Konzept:

```text
1. Trade locken
2. beide Spieler validieren
3. Items validieren
4. Geld validieren
5. Inventarplätze sichern
6. Economy sichern
7. Items entfernen
8. Geld abbuchen
9. Items geben
10. Geld gutschreiben
11. Trade als COMPLETED markieren
12. GUI schließen
```

---

# 24. Kritischer Punkt: Itemverlust

Der Server darf niemals einfach:

```text
remove items
give items
```

ohne Recovery-Mechanismus ausführen.

Vor der Transaktion müssen die Daten persistent dokumentiert werden.

Beispiel:

```text
TRADE_STARTED
TRADE_LOCKED
TRADE_VALIDATED
TRADE_ITEMS_RESERVED
TRADE_MONEY_RESERVED
TRADE_COMMIT_STARTED
TRADE_COMMIT_COMPLETED
```

---

# 25. Transaction Log

Jeder Trade erhält ein Journal.

Beispiel:

```text
trade_events
```

Events:

```text
REQUEST_CREATED
REQUEST_ACCEPTED
REQUEST_DENIED
TRADE_OPENED
ITEM_ADDED
ITEM_REMOVED
MONEY_CHANGED
PLAYER_READY
PLAYER_UNREADY
FINAL_CONFIRM_A
FINAL_CONFIRM_B
COMMIT_STARTED
COMMIT_SUCCESS
TRADE_CANCELLED
TRADE_EXPIRED
PLAYER_DISCONNECTED
RECOVERY_STARTED
RECOVERY_COMPLETED
```

---

# 26. Disconnect-Verhalten

## Spieler disconnectet vor READY

Trade wird sofort abgebrochen.

Items:

```text
zurück an Besitzer
```

Geld:

```text
unverändert
```

---

## Disconnect während Countdown

Trade wird abgebrochen.

---

## Disconnect während Finalisierung

Trade bleibt:

```text
COMMITTING
```

und Recovery übernimmt.

Der Server darf nicht einfach beide Seiten zurückgeben, ohne zu prüfen, ob die Transaktion bereits teilweise abgeschlossen wurde.

---

# 27. Serverwechsel

Wenn Spieler A von:

```text
CB01
```

nach:

```text
CB02
```

wechselt:

Standardverhalten:

```text
Trade wird abgebrochen
```

Die Items werden sicher zurückgegeben.

Optional konfigurierbar:

```yaml
cross-server-trade:
  enabled: true
  allow-server-switch: true
```

Bei aktiviertem Cross-Server-Trade darf die Session nicht an einen einzelnen Backend-Server gebunden sein.

---

# 28. Cross-Server-Trade

Beispiel:

```text
Spieler A → CB01
Spieler B → Farm01
```

Spieler A:

```text
/trade SpielerB
```

Velocity stellt fest:

```text
A = CB01
B = Farm01
```

Die Trade-ID wird zentral erzeugt.

Beide Backend-Server öffnen ihre lokale GUI.

```text
CB01
  │
  ├── TradeService
  │
  └──── Redis / Proxy ────┐
                          │
                       TradeState
                          │
  ┌───────────────────────┘
  │
Farm01
```

---

# 29. Empfohlene Cross-Server-Technik

Primär:

```text
Velocity Plugin Messaging
```

Für Echtzeit-State:

```text
Redis Pub/Sub
```

SQL:

```text
dauerhafte Speicherung
```

Nicht empfehlenswert:

```text
SQL als Echtzeit-PubSub
```

SQL ist für Persistenz gedacht, nicht für jede GUI-Aktion.

---

# 30. Redis Channels

Beispiel:

```text
trade:request
trade:response
trade:update
trade:ready
trade:confirm
trade:cancel
trade:commit
trade:recovery
trade:server
```

Payload:

```json
{
  "tradeId": "...",
  "type": "READY_CHANGED",
  "player": "...",
  "timestamp": 1759140000
}
```

---

# 31. Server Registry

Jeder Server registriert sich:

```text
server_id
server_name
online_players
last_heartbeat
status
```

Beispiel:

```text
cb01
farm01
farm02
nether
end
lobby
```

Heartbeat:

```text
alle 5 Sekunden
```

Timeout:

```text
15 Sekunden
```

---

# 32. SQL-Datenbank

Empfohlene Tabellen:

```text
trades
trade_participants
trade_items
trade_money
trade_events
trade_recovery
trade_statistics
```

---

# 33. Tabelle `trades`

Felder:

```text
id
status
created_at
updated_at
server_a
server_b
player_a
player_b
money_a
money_b
snapshot_hash
commit_started_at
completed_at
```

---

# 34. Tabelle `trade_participants`

Felder:

```text
trade_id
player_uuid
role
server
ready
confirmed
```

---

# 35. Tabelle `trade_items`

Felder:

```text
id
trade_id
player_uuid
slot
item_data
amount
item_hash
```

`item_data` muss das vollständige serialisierte Item enthalten.

---

# 36. Tabelle `trade_events`

Felder:

```text
id
trade_id
event_type
player_uuid
server
payload
created_at
```

---

# 37. Tabelle `trade_recovery`

Felder:

```text
trade_id
state
recovery_data
attempts
last_attempt
resolved_at
```

---

# 38. SQL-Transaktionen

Alle kritischen Zustände müssen über SQL-Transactions abgesichert werden.

Beispiel:

```text
BEGIN

SELECT trade FOR UPDATE

validate

update trade

insert event

COMMIT
```

Bei Fehler:

```text
ROLLBACK
```

---

# 39. Locks

Es darf immer nur eine Transaktion pro Trade laufen.

Lock-Key:

```text
trade:{tradeId}
```

Optional zusätzlich:

```text
player:{uuid}
```

Dadurch kann ein Spieler nicht gleichzeitig in zwei kritischen Trade-Prozessen verarbeitet werden.

---

# 40. GUI-Schutz

Das Plugin muss unter anderem abfangen:

```text
InventoryClickEvent
InventoryDragEvent
InventoryMoveItemEvent
InventoryPickupItemEvent
InventoryCreativeEvent
PlayerDropItemEvent
PlayerSwapHandItemsEvent
InventoryCloseEvent
PlayerQuitEvent
```

Zusätzlich müssen relevante Server-/Plugin-APIs berücksichtigt werden.

---

# 41. Inventory Close

Wenn Spieler das GUI normal schließt:

```text
Trade verlassen?
```

Optional:

```yaml
close-action: cancel
```

Standard:

```text
Trade abbrechen
```

Items werden zurückgegeben.

---

# 42. Trade-Anfragen deaktivieren

Spieler kann `/trade toggle` verwenden.

Status:

```text
Trade-Anfragen: AKTIV
```

oder:

```text
Trade-Anfragen: DEAKTIVIERT
```

---

# 43. Distanz

Bei Cross-Server-Trade:

```text
distance-check: false
```

Bei lokalem Trade:

```yaml
max-distance: 16
```

Mögliche Regel:

```text
Spieler müssen maximal 16 Blöcke entfernt sein.
```

Wenn sich ein Spieler während des Trades entfernt:

```text
Trade abbrechen
```

Optional:

```yaml
distance:
  enabled: true
  max: 32
```

---

# 44. Weltregeln

Konfigurierbar:

```yaml
disabled-worlds:
  - spawn
  - event
```

Optional:

```yaml
disabled-regions:
  - no-trade
```

Für WorldGuard:

```text
trade deny
```

---

# 45. Trade-Regionen

API:

```java
TradeRegionProvider
```

Andere Plugins können prüfen:

```java
boolean canTrade(Player player)
```

---

# 46. Nachrichten

Alle Nachrichten müssen konfigurierbar sein.

Beispiele:

```text
prefix
request-sent
request-received
request-expired
request-denied
trade-started
trade-cancelled
trade-completed
not-enough-money
player-busy
player-offline
trading-disabled
too-far-away
invalid-item
server-switch
```

---

# 47. Message-Format

MiniMessage wird empfohlen.

Beispiel:

```text
<gray>[<gold>Trade</gold>]</gray>
```

Unterstützung:

- Farben
- Hover
- Click Events
- Placeholder
- Hex-Farben

---

# 48. Placeholders

Interne Placeholder:

```text
%trade_player%
%trade_partner%
%trade_id%
%trade_money_self%
%trade_money_partner%
%trade_status%
%trade_ready_self%
%trade_ready_partner%
%trade_seconds%
```

---

# 49. PlaceholderAPI

Optional:

```text
%trade_active%
%trade_partner%
%trade_status%
%trade_money%
```

---

# 50. Sounds

Konfigurierbar:

```yaml
sounds:
  request:
  accept:
  deny:
  ready:
  unready:
  countdown:
  success:
  cancel:
  error:
```

Beispiel:

```text
ENTITY_PLAYER_LEVELUP
BLOCK_NOTE_BLOCK_PLING
ENTITY_VILLAGER_NO
BLOCK_ANVIL_LAND
```

---

# 51. Partikel

Optional:

```yaml
particles:
  enabled: true
  request: true
  ready: true
  success: true
```

Partikel dürfen niemals die Funktionalität voraussetzen.

---

# 52. Anti-Exploit

Pflichtmaßnahmen:

1. vollständige Item-Serialisierung
2. Item-Hash
3. Snapshot
4. Trade-Lock
5. Player-Lock
6. SQL-Journal
7. doppelte Validierung
8. atomarer Abschluss
9. Disconnect-Recovery
10. Serverwechsel-Recovery
11. GUI-Event-Schutz
12. Economy-Revalidierung
13. keine Client-Daten als vertrauenswürdig behandeln

---

# 53. Duplication Protection

Ein Item darf während eines Trades niemals:

```text
im Trade-Inventar
UND
im Spielerinventar
```

existieren.

Beim Hinzufügen wird es aus dem normalen Inventar in den kontrollierten Trade-State übertragen.

Die GUI ist kein echter Bukkit-Inventarbereich, der dauerhaft eine zweite Itemkopie erzeugt.

---

# 54. Empfohlene Item-Reservierung

Beim Einlegen:

```text
Spieler-Inventar
        ↓
Trade Reservation
        ↓
Trade GUI
```

Das Item gilt ab diesem Moment als reserviert.

Beim Abbruch:

```text
Trade Reservation
        ↓
Spieler-Inventar
```

Beim erfolgreichen Abschluss:

```text
Trade Reservation A
        ↓
Inventar B

Trade Reservation B
        ↓
Inventar A
```

---

# 55. Recovery

Recovery wird benötigt, wenn:

- Server abstürzt
- Proxy abstürzt
- Datenbank kurzzeitig nicht erreichbar ist
- Redis ausfällt
- Spieler disconnectet
- Economy-Operation fehlschlägt

Beim Start:

```text
TradeRecoveryManager
```

sucht nach:

```text
PENDING
OPEN
READY
CONFIRMING
COMMITTING
```

Trades.

---

# 56. Recovery-Regeln

```text
PENDING      → CANCEL
OPEN         → CANCEL
READY        → CANCEL
CONFIRMING   → CANCEL
COMMITTING   → RECOVERY REQUIRED
COMPLETED    → IGNORE
CANCELLED    → IGNORE
```

`COMMITTING` darf nicht blind abgebrochen werden.

---

# 57. Economy-Recovery

Für Geldtransaktionen sollte ein eindeutiger Transaction-Key existieren:

```text
trade:{tradeId}:money:{playerUUID}
```

Der Economy-Provider muss idealerweise idempotente Operationen unterstützen.

Beispiel:

```text
withdraw(transactionId, player, amount)
```

Wenn dieselbe Transaction-ID zweimal kommt:

```text
zweite Operation = ignorieren
```

---

# 58. API

Das Plugin soll eine öffentliche API anbieten.

Beispiel:

```java
TradeAPI
```

Methoden:

```java
requestTrade(Player sender, Player target)
openTrade(Player a, Player b)
cancelTrade(UUID tradeId)
getTrade(UUID tradeId)
isTrading(UUID player)
registerItemValidator(...)
registerTradeListener(...)
```

---

# 59. Events

API-Events:

```text
TradeRequestEvent
TradeAcceptEvent
TradeDenyEvent
TradeOpenEvent
TradeItemAddEvent
TradeItemRemoveEvent
TradeMoneyChangeEvent
TradeReadyEvent
TradeConfirmEvent
TradeCompleteEvent
TradeCancelEvent
TradeRecoveryEvent
```

---

# 60. Events müssen cancellable sein

Beispiel:

```java
TradeItemAddEvent extends Event implements Cancellable
```

Andere Plugins können dadurch Items sperren.

---

# 61. Economy-Abstraktion

Interface:

```java
public interface EconomyProvider {

    BigDecimal getBalance(UUID player);

    boolean has(UUID player, BigDecimal amount);

    boolean withdraw(UUID player, BigDecimal amount, UUID transactionId);

    boolean deposit(UUID player, BigDecimal amount, UUID transactionId);
}
```

---

# 62. Geldpräzision

Geld niemals als `double` speichern.

Nicht:

```java
double money
```

Sondern:

```java
BigDecimal
```

oder Integer in kleinster Währungseinheit.

Beispiel:

```text
100000
```

für:

```text
$100,000
```

---

# 63. Maximalbetrag

Konfigurierbar:

```yaml
economy:
  enabled: true
  max-trade-money: 1000000000000
```

---

# 64. GUI Money Input

Option A:

Chat:

```text
/trade money 50000
```

Option B:

Anvil GUI.

Empfohlen:

```text
Anvil GUI
```

Dadurch bleibt der Spieler im UI.

---

# 65. Anvil Money GUI

Titel:

```text
Geldbetrag eingeben
```

Input:

```text
50000
```

Bestätigen:

```text
[ BESTÄTIGEN ]
```

Abbrechen:

```text
ESC
```

---

# 66. Trade Summary

Vor dem Abschluss kann Slot 49 zeigen:

```text
FINAL BESTÄTIGEN

Du gibst:
• 3x Diamond
• 1x Netherite Ingot
• $50,000

Du erhältst:
• 2x Emerald Block
• $10,000
```

---

# 67. Erfolgreicher Trade

Nach Erfolg:

```text
<green>Trade erfolgreich abgeschlossen!</green>
```

GUI:

```text
Trade abgeschlossen
```

Sound:

```text
ENTITY_PLAYER_LEVELUP
```

---

# 68. Abgebrochener Trade

```text
<red>Der Trade wurde abgebrochen.</red>
```

Items werden zurückgegeben.

---

# 69. Fehlgeschlagene Transaktion

Wenn etwas schiefgeht:

```text
Trade konnte nicht abgeschlossen werden.
Die Transaktion wurde gesichert.
```

Intern:

```text
RECOVERY_REQUIRED
```

Admin erhält optional:

```text
Trade TRD-... benötigt Recovery.
```

---

# 70. YAML-Dateien

Benötigte Dateien – nur Dateinamen:

```text
config.yml
messages.yml
gui.yml
sounds.yml
items.yml
economy.yml
permissions.yml
database.yml
redis.yml
proxy.yml
servers.yml
worlds.yml
regions.yml
settings.yml
logging.yml
recovery.yml
limits.yml
placeholders.yml
api.yml
```

Optional:

```text
menus.yml
blocked-items.yml
debug.yml
metrics.yml
```

---

# 71. `config.yml`

Hauptkonfiguration.

Enthält:

```text
enable
debug
language
database
redis
economy
cross-server
requests
limits
distance
worlds
regions
recovery
logging
```

---

# 72. `messages.yml`

Alle Spielertexte.

---

# 73. `gui.yml`

GUI:

```text
title
size
slots
materials
display-names
lore
```

---

# 74. `sounds.yml`

Alle Sounds.

---

# 75. `items.yml`

Item-Regeln:

```text
blocked
allowed
special handling
```

---

# 76. `economy.yml`

Economy:

```text
provider
max amount
minimum amount
format
decimal places
```

---

# 77. `database.yml`

SQL:

```text
host
port
database
username
password
pool-size
timeouts
ssl
```

Passwörter nicht in Git committen.

---

# 78. `redis.yml`

Redis:

```text
host
port
database
password
channels
timeouts
```

---

# 79. `proxy.yml`

Proxy:

```text
enabled
mode
channel
timeout
server-switch
```

---

# 80. `servers.yml`

Server Registry:

```text
server-id
display-name
trade-enabled
```

---

# 81. `worlds.yml`

Trade-Regeln pro Welt.

---

# 82. `regions.yml`

Trade-Regeln pro Region.

---

# 83. `logging.yml`

Logging:

```text
trade-created
trade-completed
trade-cancelled
admin-actions
recovery
errors
```

---

# 84. `recovery.yml`

Recovery-Einstellungen:

```text
enabled
interval
max-attempts
timeout
auto-recover
```

---

# 85. `limits.yml`

Limits:

```text
request cooldown
trade timeout
max items
max money
max active trades
```

---

# 86. Paketstruktur

Empfohlene Java-Struktur:

```text
de.opwolfii.trade
├── TradePlugin
│
├── command
│   ├── TradeCommand
│   └── TradeAdminCommand
│
├── trade
│   ├── Trade
│   ├── TradeManager
│   ├── TradeState
│   ├── TradeRequest
│   ├── TradeSnapshot
│   ├── TradeValidator
│   ├── TradeLockManager
│   └── TradeRecoveryManager
│
├── gui
│   ├── TradeGUI
│   ├── TradeGUIListener
│   └── MoneyInputGUI
│
├── item
│   ├── ItemSerializer
│   ├── ItemHash
│   ├── ItemReservation
│   └── ItemValidator
│
├── economy
│   ├── EconomyProvider
│   └── VaultEconomyProvider
│
├── database
│   ├── DatabaseManager
│   ├── TradeRepository
│   ├── EventRepository
│   └── RecoveryRepository
│
├── proxy
│   ├── ProxyBridge
│   ├── RedisBridge
│   └── ServerRegistry
│
├── api
│   ├── TradeAPI
│   └── events
│
├── listener
│
├── config
│
└── util
```

---

# 87. Trade-State-Machine

```text
                    ┌───────────────┐
                    │    PENDING    │
                    └───────┬───────┘
                            │ accept
                            ▼
                    ┌───────────────┐
                    │     OPEN      │
                    └───────┬───────┘
                            │ both ready
                            ▼
                    ┌───────────────┐
                    │     READY     │
                    └───────┬───────┘
                            │ countdown
                            ▼
                    ┌───────────────┐
                    │  CONFIRMING   │
                    └───────┬───────┘
                            │ both confirm
                            ▼
                    ┌───────────────┐
                    │  COMMITTING   │
                    └───────┬───────┘
                            │ success
                            ▼
                    ┌───────────────┐
                    │   COMPLETED   │
                    └───────────────┘

OPEN / READY / CONFIRMING
        │
        ├── cancel
        ├── disconnect
        ├── timeout
        └── invalid state
                ↓
           CANCELLED
```

---

# 88. Timeout

Empfohlene Standardwerte:

```text
Request: 30 Sekunden
Trade: 10 Minuten
Confirmation: 15 Sekunden
```

Bei Ablauf:

```text
Trade CANCELLED
```

---

# 89. Anti-Spam

Trade-Anfrage:

```text
3 Sekunden Cooldown
```

Zusätzlich:

```text
max 5 requests / 30 Sekunden
```

Bei Missbrauch:

```text
temporary request block
```

---

# 90. Concurrent Trade Protection

Ein Spieler darf niemals gleichzeitig:

```text
Trade A
Trade B
```

haben.

Serverübergreifend muss dieser Lock zentral sein.

Beispiel:

```text
player:{uuid}:trade-lock
```

---

# 91. GUI-Click Protection

Wenn Slot nicht explizit als Spieler-Trade-Slot definiert ist:

```text
event.setCancelled(true)
```

Ausnahmen:

```text
eigene Item-Slots
Money-Slot
Ready
Cancel
Confirm
```

---

# 92. Shift-Klick

Shift-Klick aus dem Spielerinventar:

```text
→ nächster freier eigener Trade-Slot
```

Wenn voll:

```text
Alle Trade-Slots sind voll.
```

---

# 93. Dragging

Nur eigene Trade-Slots dürfen Items aufnehmen.

Fremde Trade-Slots:

```text
BLOCK
```

---

# 94. Hotbar Swap

Number-Key auf fremden Slots:

```text
BLOCK
```

Number-Key auf eigenen Trade-Slot:

```text
BLOCK
```

Da Trade-Slots nicht mit normalen Inventarplätzen vertauscht werden sollen.

---

# 95. Double Click

Doppelklick darf nicht verwendet werden, um Items aus fremden Bereichen einzusammeln.

---

# 96. Cursor Item

Beim Schließen muss auch das Cursor-Item berücksichtigt werden.

Es darf kein Item:

```text
Cursor
```

sein und anschließend verloren gehen.

---

# 97. Server Crash während GUI

Recovery über:

```text
trade_items
trade_events
trade_recovery
```

---

# 98. Server Crash nach Item-Removal

Wenn:

```text
Items entfernt
```

aber:

```text
Items noch nicht übertragen
```

wurde:

```text
RECOVERY_REQUIRED
```

Recovery gibt die Items entsprechend dem Journal zurück oder vollzieht den noch fehlenden Commit.

---

# 99. Audit Logging

Jeder Trade muss nachvollziehbar sein.

Admin kann sehen:

```text
Trade ID
Zeit
Spieler A
Spieler B
Server
Items
Money
Events
Result
```

---

# 100. Datenschutz

Nur für den technischen Zweck notwendige Daten speichern.

Keine unnötigen personenbezogenen Daten.

UUID statt Klartext-Accountdaten verwenden.

Logs sollten konfigurierbar sein.

---

# 101. Performance

Das System soll bei einem Netzwerk mit vielen Spielern funktionieren.

Wichtig:

- kein SQL-Query bei jedem InventoryClick
- kein synchroner Netzwerkzugriff im Main Thread
- Redis asynchron
- SQL asynchron
- Datenbank-Pooling
- Trade-State im RAM
- nur kritische Zustände persistent schreiben
- Batch-/Prepared-Statements

---

# 102. Cache

RAM:

```text
activeTrades
activeRequests
playerLocks
serverRegistry
```

Redis:

```text
distributed locks
cross-server state
events
```

SQL:

```text
persistent source
```

---

# 103. Threading

Minecraft-Bukkit/Paper-API:

```text
Main Thread
```

SQL:

```text
Async
```

Redis:

```text
Async
```

Wenn ein Async-Callback Bukkit-Daten benötigt:

```text
zurück auf Main Thread
```

---

# 104. Reload

`/tradeadmin reload` darf niemals aktive Trades zerstören.

Beim Reload:

```text
Config reload
Messages reload
GUI definitions reload
```

Nicht:

```text
TradeManager komplett neu erstellen
```

Aktive Trades bleiben erhalten.

---

# 105. Plugin Disable

Bei Plugin-Shutdown:

1. neue Trades verhindern
2. aktive Trades sauber canceln
3. Items zurückgeben
4. States persistieren
5. Redis schließen
6. SQL Pool schließen

---

# 106. Proxy Shutdown

Bei Proxy-Shutdown müssen Backend-Server erkennen:

```text
proxy unavailable
```

Cross-Server-Trades werden entweder:

```text
pause
```

oder:

```text
cancel
```

Empfohlen:

```text
cancel + recovery-safe
```

---

# 107. Redis-Ausfall

Wenn Redis ausfällt:

```text
Cross-server trades deaktivieren
```

Lokale Trades können optional weiterlaufen.

Spieler erhalten:

```text
Cross-Server-Trading ist momentan nicht verfügbar.
```

---

# 108. SQL-Ausfall

Wenn SQL nicht erreichbar ist:

```text
Keine neuen Trades
```

Bestehende Trades:

```text
nicht committen
```

Es darf kein Trade ohne funktionierende Persistenz abgeschlossen werden.

---

# 109. Economy-Ausfall

Wenn Economy nicht erreichbar ist:

```text
Money-Trades deaktivieren
```

Item-only Trades können optional weiterlaufen.

Konfiguration:

```yaml
economy:
  fail-mode: disable-money-trades
```

---

# 110. Item-only Trade

Money kann deaktiviert werden.

Dann wird Slot 27/35:

```text
nicht angezeigt
```

oder:

```text
Geldhandel deaktiviert
```

---

# 111. Nur-Geld-Trade

Wenn keine Items enthalten sind:

```text
Spieler A: $100,000
Spieler B: $50,000
```

Netto:

```text
A → +50,000
B → -50,000
```

Es darf nicht zuerst blind abgebucht werden, ohne die Gegenoperation abzusichern.

---

# 112. Netto-Berechnung

Optional kann das System optimieren:

```text
A gibt 100.000
B gibt 50.000
```

statt:

```text
A -100.000
B +100.000
B -50.000
A +50.000
```

direkt:

```text
A -50.000
B +50.000
```

Aber nur wenn der Economy-Provider diese Operationen sauber unterstützt.

Für maximale Nachvollziehbarkeit kann alternativ die vollständige Brutto-Transaktion verwendet werden.

---

# 113. Trade History

Optional:

```text
/trade history
```

Zeigt:

```text
letzte Trades
Partner
Datum
Status
```

Keine vollständigen Itemdaten im Spieler-GUI notwendig.

---

# 114. Trade Statistics

Optional:

```text
Trades gestartet
Trades abgeschlossen
Trades abgebrochen
Gesamter Money-Umsatz
```

Diese Statistiken dürfen nicht zur Spielmechanik benötigt werden.

---

# 115. Admin GUI

Optional:

```text
/tradeadmin gui
```

Anzeigen:

```text
aktive Trades
hängende Trades
Recovery
Fehler
```

---

# 116. Security Logging

Besonders protokollieren:

```text
mehrfaches Commit
ungültiger Snapshot
Item Hash mismatch
Money mismatch
Player Lock violation
Trade Lock violation
Recovery
```

---

# 117. Debug-Modus

Debug-Logs:

```text
[Trade] Request created
[Trade] Trade opened
[Trade] Item added
[Trade] Snapshot created
[Trade] Validation successful
[Trade] Commit started
[Trade] Commit completed
```

Kein Debug-Spam im normalen Betrieb.

---

# 118. Konfigurierbare Trade-Modi

Optional:

```yaml
modes:
  items: true
  money: true
  cross-server: true
```

Später erweiterbar:

```text
commands
tokens
auction-items
custom currencies
```

---

# 119. Custom Currency API

Andere Plugins können später Währungen registrieren:

```java
TradeCurrencyProvider
```

Beispiele:

```text
Money
Coins
Tokens
Kristalle
```

Jede Währung erhält:

```text
id
displayName
icon
balance()
withdraw()
deposit()
```

---

# 120. GUI-Currency-Slots

Erweiterbar:

```text
Money → 27 / 35
Currency 2 → optional
Currency 3 → optional
```

Das Basissystem sollte jedoch zunächst nur eine Economy unterstützen.

---

# 121. Permissions pro Feature

Beispiele:

```text
trade.use
trade.items
trade.economy
trade.crossserver
trade.history
trade.toggle
```

Dadurch können bestimmte Ränge Funktionen erhalten.

---

# 122. Beispiel-Rangkonzept

Spieler:

```text
trade.use
trade.items
trade.economy
```

VIP:

```text
trade.history
```

Staff:

```text
trade.admin.*
```

---

# 123. PlaceholderAPI-Integration

Optionales Modul:

```text
TradeExpansion
```

Expansion:

```text
trade
```

Beispiele:

```text
%trade_active%
%trade_partner%
%trade_status%
%trade_money_self%
%trade_money_partner%
```

---

# 124. TAB-Integration

Das System kann optional Placeholder für TAB bereitstellen:

```text
%trade_status%
```

Beispiel:

```text
⚔ Trade mit SpielerX
```

---

# 125. Discord Integration

Optionales externes Modul.

Trade-Events können an Discord gesendet werden.

Nicht standardmäßig nötig.

Beispiel:

```text
Trade completed
Spieler A ↔ Spieler B
```

Keine sensiblen Daten loggen.

---

# 126. API-Versionierung

API:

```text
v1
```

Pakete:

```text
de.opwolfii.trade.api.v1
```

Bei Breaking Changes:

```text
v2
```

---

# 127. Maven-Abhängigkeiten

Empfohlen:

```text
Paper API
Velocity API
HikariCP
JDBC Driver
Redis Client
Adventure API
PlaceholderAPI optional
Vault optional
```

---

# 128. Datenbanktreiber

Je nach DB:

```text
MySQL / MariaDB
```

Empfohlen:

```text
HikariCP
```

Connection Pool:

```text
minimum-idle: 5
maximum-pool-size: 20
```

Bei hoher Last anpassen.

---

# 129. SQL Isolation

Für kritische Trades:

```text
transaction isolation
```

entsprechend der verwendeten DB konfigurieren.

Ziel:

```text
keine doppelten Commits
```

---

# 130. Idempotenz

Alle kritischen Operationen müssen eine ID besitzen.

Beispiel:

```text
commit:{tradeId}
withdraw:{tradeId}:{player}
deposit:{tradeId}:{player}
```

Wird derselbe Vorgang erneut ausgeführt:

```text
kein zweiter Effekt
```

---

# 131. Race Condition Beispiel

Problem:

```text
A bestätigt
B bestätigt

Thread 1: commit
Thread 2: commit
```

Lösung:

```text
distributed trade lock
```

Nur ein Thread darf:

```text
COMMITTING
```

setzen.

---

# 132. Double Confirmation

Auch:

```text
B bestätigt
B bestätigt erneut
```

muss idempotent sein.

Keine doppelte Aktion.

---

# 133. GUI Titel

Standard:

```text
<dark_gray>Trade mit <gold>%partner%</gold>
```

---

# 134. Item-Lore

Eigenes Item:

```text
<gray>Dein Angebot
```

Fremdes Item:

```text
<gray>Angebot von <yellow>%partner%</yellow>
```

Geld:

```text
<gold>Geldangebot</gold>

<gray>Betrag:
<green>$50,000
```

---

# 135. Ready Button

Nicht bereit:

```text
<red>NICHT BEREIT
```

Lore:

```text
<gray>Klicke, um dein Angebot zu bestätigen.
```

Bereit:

```text
<green>BEREIT
```

Lore:

```text
<gray>Warte auf den anderen Spieler.
```

---

# 136. Final Button

Vor Ready:

```text
<gray>Warte auf beide Spieler.
```

Nach Ready:

```text
<yellow>FINAL BESTÄTIGEN
```

---

# 137. Cancel Button

```text
<red>TRADE ABBRECHEN
```

Lore:

```text
<gray>Bricht den gesamten Trade ab.
```

---

# 138. GUI-Status

Slot 4:

```text
Spieler A: READY
```

Slot 13:

```text
Spieler B: READY
```

Slot 22:

```text
Trade-Status
```

Slot 31:

```text
Geldstatus
```

Slot 40:

```text
Countdown
```

---

# 139. Accessibility

Nicht nur Farben verwenden.

READY sollte zusätzlich:

```text
BEREIT
```

anzeigen.

Nicht nur:

```text
grün
```

---

# 140. Localization

Mindestens:

```text
de
en
```

Dateien:

```text
messages_de.yml
messages_en.yml
```

Optional:

```text
messages.yml
```

als aktive Sprachdatei.

---

# 141. Konfigurationssprache

Standard:

```text
de
```

---

# 142. Testsystem

Vor Release müssen mindestens folgende Tests bestehen:

```text
normal trade
item-only trade
money-only trade
item + money trade
cancel
timeout
disconnect
server switch
server crash
proxy restart
database outage
redis outage
economy outage
duplicate click
shift click
drag
hotbar swap
double click
creative inventory
```

---

# 143. Exploit-Test

Testen:

```text
Item während Commit
Disconnect während Commit
Serverwechsel während Commit
Reload während Trade
Plugin Disable während Trade
Proxy Disconnect
Redis Disconnect
DB Disconnect
Money Change während Confirmation
```

---

# 144. Belastungstest

Simulieren:

```text
100 aktive Trades
500 aktive Trades
1000 aktive Trades
```

Ziel:

Kein signifikanter TPS-Einbruch.

---

# 145. Logging-Level

```text
ERROR
WARN
INFO
DEBUG
TRACE
```

Standard:

```text
INFO
```

---

# 146. Metriken

Optional:

```text
trade_requests_total
trade_completed_total
trade_cancelled_total
trade_failed_total
trade_recovery_total
trade_commit_duration
trade_active
```

---

# 147. Monitoring

Bei Problemen sollte erkennbar sein:

```text
Wie viele Trades aktiv?
Wie viele hängen?
Wie viele Recovery-Fälle?
Wie lange dauern Commits?
```

---

# 148. Fehlercodes

Beispiele:

```text
TRD-001 PLAYER_OFFLINE
TRD-002 PLAYER_BUSY
TRD-003 REQUEST_EXPIRED
TRD-004 INVALID_ITEM
TRD-005 INSUFFICIENT_FUNDS
TRD-006 SERVER_UNAVAILABLE
TRD-007 DATABASE_ERROR
TRD-008 ECONOMY_ERROR
TRD-009 SNAPSHOT_MISMATCH
TRD-010 COMMIT_ERROR
TRD-011 RECOVERY_REQUIRED
```

---

# 149. Sicherheitsprinzip

Der Client ist niemals vertrauenswürdig.

Das GUI liefert nur Eingaben.

Server entscheidet:

```text
Ist das Item wirklich vorhanden?
Ist der Spieler wirklich berechtigt?
Ist genug Geld vorhanden?
Ist der Trade noch gültig?
Ist der Snapshot unverändert?
```

---

# 150. Trade-Flow vollständig

## Schritt 1

A:

```text
/trade B
```

## Schritt 2

System prüft:

```text
A online
B online
A != B
A nicht im Trade
B nicht im Trade
Trade-Anfragen erlaubt
Cooldown
Server verfügbar
```

## Schritt 3

Request erstellen.

## Schritt 4

B akzeptiert.

## Schritt 5

Trade-ID erstellen.

## Schritt 6

Trade-State speichern.

## Schritt 7

GUI öffnen.

## Schritt 8

Spieler legen Items ein.

## Schritt 9

Money optional setzen.

## Schritt 10

A READY.

## Schritt 11

B READY.

## Schritt 12

Countdown.

## Schritt 13

Snapshot.

## Schritt 14

Final confirmation.

## Schritt 15

Trade Lock.

## Schritt 16

Validation.

## Schritt 17

Commit.

## Schritt 18

Items übertragen.

## Schritt 19

Money übertragen.

## Schritt 20

COMPLETED.

## Schritt 21

GUI schließen.

## Schritt 22

Erfolg anzeigen.

---

# 151. Abbruch-Flow

Bei:

```text
Cancel
Disconnect
Timeout
Server switch
Invalid state
```

→ Lock

→ Snapshot/Reservation laden

→ Items zurückgeben

→ Geld unverändert lassen

→ Trade als CANCELLED markieren

→ GUI schließen

→ Spieler informieren.

---

# 152. Priorität der Daten

Bei Konflikten gilt:

```text
1. Trade Transaction Journal
2. SQL Trade State
3. serverseitiger Trade State
4. GUI
5. Client
```

Das GUI ist niemals die Quelle der Wahrheit.

---

# 153. Quelle der Wahrheit

Für aktive Trades:

```text
TradeService / zentraler State
```

Für dauerhafte Transaktionen:

```text
SQL
```

Für Items:

```text
serverseitig reservierte Itemdaten
```

Für UI:

```text
GUI
```

---

# 154. Empfehlung für das Netzwerk

Bei einem größeren Netzwerk:

```text
Velocity
    │
    ├── Trade Proxy Module
    │
    ├── Redis
    │
    └── MySQL/MariaDB
             │
      ┌──────┼─────────┐
      │      │         │
     CB01   Farm01    End
      │      │         │
      └── Trade Plugin┘
```

Jeder Backend-Server besitzt das gleiche Trade-Plugin.

---

# 155. Was NICHT gemacht werden sollte

Nicht:

```text
Trade-Daten nur im RAM speichern
```

Nicht:

```text
Trade nur über GUI verwalten
```

Nicht:

```text
Items per Material speichern
```

Nicht:

```text
Money als double
```

Nicht:

```text
Commit ohne Lock
```

Nicht:

```text
SQL synchron im Main Thread
```

Nicht:

```text
Serverwechsel ignorieren
```

Nicht:

```text
Disconnect während Commit ignorieren
```

---

# 156. Erweiterbarkeit

Das System sollte später problemlos erweitert werden können um:

```text
/market
/auctionhouse
/barter
/confirm
/custom currency
/escrow
/trade history
/trade requests
```

Das Trade-System sollte deshalb als eigenständiger Service entwickelt werden.

---

# 157. Empfohlene Module

```text
trade-core
trade-paper
trade-velocity
trade-api
trade-economy
trade-database
trade-redis
trade-placeholderapi
```

Bei einem kleineren Projekt kann alles in einem Plugin bleiben.

---

# 158. Minimalinstallation

Für eine einfache Installation:

```text
Trade Plugin
MySQL/MariaDB
Velocity
Economy Provider
```

Redis ist für ein echtes Multi-Server-System sehr empfehlenswert.

---

# 159. Produktionsinstallation

Empfohlen:

```text
Velocity
Trade Proxy Module
Trade Backend Plugin
MariaDB/MySQL
Redis
LuckPerms
Economy
PlaceholderAPI optional
WorldGuard optional
```

---

# 160. Produktions-Sicherheitscheck

Vor Livebetrieb:

```text
[ ] SQL funktioniert
[ ] Redis funktioniert
[ ] Economy funktioniert
[ ] Trade Lock funktioniert
[ ] Recovery funktioniert
[ ] Disconnect getestet
[ ] Serverwechsel getestet
[ ] Crash getestet
[ ] Duplication Tests bestanden
[ ] Money Tests bestanden
[ ] GUI Tests bestanden
[ ] Permission Tests bestanden
[ ] 100+ parallele Trades getestet
```

---

# 161. Definition of Done

Das Trade-System gilt erst als fertig, wenn:

- `/trade <player>` funktioniert
- Requests funktionieren
- GUI funktioniert
- Items funktionieren
- Money funktioniert
- Ready funktioniert
- Confirmation funktioniert
- Cancel funktioniert
- Timeout funktioniert
- Disconnect funktioniert
- Serverwechsel sicher behandelt wird
- Proxy-Kommunikation funktioniert
- Datenbank persistiert
- Recovery funktioniert
- Dupe-Tests bestanden sind
- Economy-Fehler behandelt werden
- SQL-Fehler behandelt werden
- Redis-Fehler behandelt werden
- API funktioniert
- Permissions funktionieren
- Messages konfigurierbar sind
- GUI konfigurierbar ist
- Sounds konfigurierbar sind
- Logs funktionieren

---

# 162. Kurzfassung der finalen Architektur

```text
                       VELOCITY
                          │
                    Trade Proxy
                          │
                ┌─────────┴─────────┐
                │                   │
              Redis               SQL
                │                   │
                │             Persistent State
                │
       ┌────────┴────────┐
       │                 │
     CB01              FARM01
       │                 │
       └──── Trade Plugin┘
               │
             GUI
               │
         Trade Service
               │
      ┌────────┴────────┐
      │                 │
    Items             Money
      │                 │
 Reservation       Economy API
```

---

# 163. Wichtigste Designentscheidung

Der wichtigste Punkt der gesamten Implementierung ist:

**Ein Trade ist keine einfache Inventar-GUI, sondern eine transaktionale Operation.**

Das GUI ist lediglich die Benutzeroberfläche.

Die eigentliche Wahrheit liegt in:

```text
Trade State
+
Item Reservation
+
Money State
+
Snapshot
+
Transaction Journal
+
Locks
+
Recovery
```

Dadurch kann das System auch mit:

- Serverabstürzen
- Proxy-Neustarts
- Disconnects
- Netzwerkproblemen
- hoher Spielerzahl
- mehreren Backend-Servern

sicher umgehen.

---

# 164. Endgültige Dateiliste

## Plugin-Dateien

```text
plugin.yml
config.yml
messages.yml
gui.yml
sounds.yml
items.yml
economy.yml
permissions.yml
database.yml
redis.yml
proxy.yml
servers.yml
worlds.yml
regions.yml
settings.yml
logging.yml
recovery.yml
limits.yml
placeholders.yml
api.yml
```

## Sprachdateien

```text
messages_de.yml
messages_en.yml
```

## Optional

```text
menus.yml
blocked-items.yml
debug.yml
metrics.yml
```

---

# 165. Schluss

Dieses Konzept beschreibt ein vollständiges, serverübergreifendes `/trade`-System für ein größeres Minecraft-Netzwerk.

Die Implementierung soll sich an folgenden Grundsätzen orientieren:

```text
SICHERHEIT
│
├── keine Duplikation
├── kein Itemverlust
├── kein Moneyverlust
├── atomare Transaktionen
├── Locks
├── Snapshots
└── Recovery

MULTI-SERVER
│
├── Velocity
├── Redis
├── SQL
└── Backend Trade Plugin

BENUTZERERLEBNIS
│
├── einfache Anfrage
├── übersichtliches GUI
├── klare Statusanzeige
├── Ready
├── Countdown
└── Final Confirm

ENTWICKLER
│
├── API
├── Events
├── Provider
├── Konfiguration
├── Logging
└── Erweiterbarkeit
```

**Das System soll nicht nur funktionieren, wenn alles normal läuft, sondern insbesondere dann korrekt reagieren, wenn etwas schiefgeht.**
