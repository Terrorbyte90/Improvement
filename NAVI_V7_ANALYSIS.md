# Navi-v7 Analys — Komplett Buggrapport och Förbättringsförslag

**Analys utförd:** 2026-04-02  
**Analytiker:** Navi Workers (5 parallella)  
**Version:** Navi-v7 (GitHub)

---

## Sammanfattning

Analyserade hela Navi-v7-kodbasen med 5 parallella workers:
- Orchestrator (orchestrator.js, 52KB)
- Workers (worker.js, worker-pool.js)
- Push/Notiser (push-service.js)
- Server & WebSocket (server.js)
- Verktyg (tools/ ~45+ verktyg)

**Totalt funna:**
- 🔴 Kritsika buggar: 12
- 🟠 Högprioriterade buggar: 8
- 🟡 Medelprioriterade buggar: 15
- 🟢 Förbättringsförslag: 25

---

## Innehåll

1. [Orchestrator](#1-orchestrator)
2. [Workers](#2-workers)
3. [Push/Notiser](#3-pushnotiser)
4. [Server & WebSocket](#4-server--websocket)
5. [Verktyg](#5-verktyg)
6. [Åtgärdslista](#6-åtgärdslista)

---

## 1. Orchestrator

**Fil:** `/root/navi-v7/orchestrator.js` (51,999 bytes)

### 🔴 Kritsika Buggar

| # | Rad | Beskrivning |
|---|-----|-------------|
| 1 | ~280-285 | **Rate limit retry utan iterCount-ökning** — kan leda till oändlig loop |
| 2 | ~215 | **Ingen felhantering för memory.appendMessage** — kan blockera hela chatten |
| 3 | ~330-340 | **_detectWorkerRequest kan krascha på null text** |
| 4 | ~390-410 | **Felaktig LRU-cache-implementation** — rensar godtycklig post |
| 5 | ~580-595 | **Session cleanup race condition** — kan ta bort busy-session |
| 6 | ~590-595 | **workerJobMap cleanup för tidigt** — tar bort fortfarande körande jobb |

### 🟡 Ineffektiviteter

| # | Rad | Beskrivning |
|---|-----|-------------|
| 1 | ~30 | classifyTaskComplexity för bred regex — nästan alla förfrågor = 'complex' |
| 2 | ~440-530 | Lång system-prompt varje anrop — ingen trunkering |
| 3 | ~220-240 | _preReadCodeContext läser filer även för enkla frågor |
| 4 | ~405 | Cache-nyckel trunkeras till 80 tecken — risk för kollisioner |

### 🟢 Förbättringsförslag

1. Fixa rate limit-loop (öka iterCount vid retry)
2. Lägg till try/catch kring memory.appendMessage
3. Verifiera att _autoLearnFromConversation existerar
4. Implementera riktig LRU-cache med timestamp
5. Kontrollera busy-status innan session cleanup
6. Förbättra classifyTaskComplexity regex

---

## 2. Workers

**Filer:** `worker.js` (27KB), `worker-pool.js` (9KB)

### 🔴 Kritsika Buggar

| # | Fil | Rad | Beskrivning |
|---|-----|-----|-------------|
| 1 | worker-pool.js | 210 | **Worker tas inte bort efter watchdog** — minnesläcka |
| 2 | worker-pool.js | 200 | **setInterval utan cleanup** — multipla intervals vid omstart |
| 3 | worker.js | 90 | **Message pushas före verktygsresultat** — ofullständig historik |
| 4 | worker.js | 145 | **break i verktygsloopen påverkar inte huvudloopen** |

### 🟠 Högprioriterade Buggar

| # | Fil | Rad | Beskrivning |
|---|-----|-----|-------------|
| 5 | worker.js | 74 | För aggressiv backoff (30s steg vid rate limits) |
| 6 | worker-pool.js | 44 | Race condition vid concurrent dequeue |
| 7 | worker.js | 119 | Tyst JSON-parse-fel — args blir {} |
| 8 | worker.js | 127 | Returnerar error-objekt hanteras inte |

### 🟡 Medelprioriterade Buggar

| # | Fil | Rad | Beskrivning |
|---|-----|-----|-------------|
| 9 | worker.js | 60 | onProgress anropas med fel status |
| 10 | worker.js | 158 | Svag stuck-detektering (bara 100 tecken) |
| 11 | worker.js | 117 | Saknar timeout för verktygsanrop |
| 12 | worker-pool.js | 6 | Hårdkodad MAX_WORKERS = 20 |
| 13 | worker.js | 58 | Trim note kan vara felaktig |
| 14 | worker.js | 10 | HISTORY_TRIM_THRESHOLD = 60 för högt |

### 🟢 Förbättringsförslag

1. Lägg till `this.workers.delete(jobId)` efter watchdog timeout
2. Flytta `this.history.push(message)` till EFTER verktygsresultat
3. Lås `_dequeueNext()` med mutex
4. Ändra rate limit backoff till 10s steg
5. Spara interval-ID och rensa vid shutdown
6. Lägg till worker-prioritet och pausering

---

## 3. Push/Notiser

**Fil:** `push-service.js`

### 🔴 Kritsika Buggar

| # | Rad | Beskrivning |
|---|-----|-------------|
| 1 | 46-62 | **Ingen retry-logik** — vid timeout/503 görs inga återförsök |
| 2 | 26 | **Ingen persistent token-lagring** — tokens försvinner vid omstart |
| 3 | - | **Ingen APNS-återanslutning** — vid anslutningsfel ingen återhämtning |

### 🟠 Högprioriterade Buggar

| # | Rad | Beskrivning |
|---|-----|-------------|
| 4 | 26-28 | Ingen token-validering — alla strängar accepteras |
| 5 | 52 | Endast första felet hanteras (result.failed[0]) |
| 6 | 53-55 | Token-borttagning för aggressiv — tas bort vid EN failed-notis |

### 🟢 Förbättringsförslag

1. Lägg till exponential backoff retry
2. Spara tokens till fil/Redis
3. Validera token-format (64 hex-tecken)
4. Lägg till worker_started, worker_queued notiser
5. Logga alla misslyckanden ordentligt

---

## 4. Server & WebSocket

**Fil:** `server.js` (16,616 bytes)

### 🔴 Kritsika Buggar

| # | Rad | Beskrivning |
|---|-----|-------------|
| 1 | 25 | **Hårdkodad API_KEY fallback** — 'navi-brain-2026' |
| 2 | 155-158 | **broadcast() saknar felhantering** — kan krascha vid nätverksfel |
| 3 | 187-190 | **JSON.parse utan storlekskontroll** — DoS-risk |

### 🟠 Högprioriterade Buggar

| # | Rad | Beskrivning |
|---|-----|-------------|
| 4 | 185-260 | Ingen rate limiting på WebSocket |
| 5 | 195-198 | sessionId utan validering |
| 6 | 199-225 | Klient kan försvinna under async CHAT |

### 🟡 Medelprioriterade Buggar

| # | Rad | Beskrivning |
|---|-----|-------------|
| 7 | 271 | Dubbel borttagning i ping-loop |
| 8 | 89-108 | onWorkerDone bygger stora strängar utan begränsning |
| 9 | ~130-150 | Ingen felhantering i express routes |

### 🟢 Säkerhetsproblem

| # | Rad | Beskrivning |
|---|-----|-------------|
| 1 | 25 | API_KEY fallback — kräv att variabeln finns |
| 2 | 185-260 | Rate limiting saknas |
| 3 | 195-230 | Session hijacking risk |

### 🟢 Förbättringsförslag

1. Kräv API_KEY i .env, avsluta om saknas
2. Lägg till rate limiting per klient
3. Validera sessionId (max 100 tecken)
4. Lägg till meddelandestorlekskontroll (1MB max)
5. Minska ping-intervall till 15s
6. Öka graceful shutdown timeout till 30s

---

## 5. Verktyg

**Katalog:** `/root/navi-v7/tools/` (~45+ verktyg)

### 🔴 Kritsika Säkerhetsproblem

| # | Fil | Beskrivning |
|---|-----|-------------|
| 1 | filesystem.js | **Path traversal** — ingen sökvägsvalidering |
| 2 | exec.js | **Bred shell-whitelist** — python3, sed, curl tillåtna |
| 3 | server-control.js | **Obgränsad server-kontroll** — alla PM2-processer |
| 4 | github.js | **GitHub push utan granskning** — kan pusha valfri kod |

### 🟠 Högprioriterade Problem

| # | Fil | Beskrivning |
|---|-----|-------------|
| 5 | Alla verktyg | Ingen rate limiting |
| 6 | Alla verktyg | Inget audit-loggning |
| 7 | Verktyg | Felmeddelanden kan läcka information |

### 🟡 Medelprioriterade Problem

| # | Fil | Beskrivning |
|---|-----|-------------|
| 8 | github.js | Inkonsekvent felhantering |
| 9 | memory-tools.js | Kräver init() annars fungerar ej |
| 10 | filesystem.js | read_multiple_files saknar element-validering |
| 11 | search.js | HTML-stripping ofullständig |

### 🟢 Förbättringsförslag

1. **Lägg till sökvägsvalidering:**
```javascript
const ALLOWED_BASE_PATHS = ['/root/navi-v7', '/root/repos'];
function isPathAllowed(filePath) {
  const resolved = path.resolve(filePath);
  return ALLOWED_BASE_PATHS.some(base => resolved.startsWith(base));
}
```

2. **Begränsa exec-whitelist** — ta bort python3, sed, curl

3. **Vitlista PM2-processer:**
```javascript
const ALLOWED_PM2_PROCESSES = ['navi-v7'];
```

4. **Lägg till rate limiting per verktyg**

5. **Lägg till audit-loggning**

---

## 6. Åtgärdslista

### Omedelbart (idag)

| Prioritet | Komponent | Åtgärd |
|-----------|-----------|--------|
| 🔴 | Server | Ta bort API_KEY fallback, kräv .env |
| 🔴 | Verktyg | Lägg till path traversal-skydd |
| 🔴 | Verktyg | Begränsa exec-whitelist |
| 🔴 | Push | Lägg till persistent token-lagring |
| 🔴 | Worker | Fix minnesläcka efter watchdog |

### Denna vecka

| Prioritet | Komponent | Åtgärd |
|-----------|-----------|--------|
| 🟠 | Orchestrator | Fixa rate limit retry-loop |
| 🟠 | Workers | Lås _dequeueNext() med mutex |
| 🟠 | Server | Lägg till rate limiting |
| 🟠 | Push | Lägg till retry-logik |
| 🟠 | Verktyg | Lägg till rate limiting |

### Nästa vecka

| Prioritet | Komponent | Åtgärd |
|-----------|-----------|--------|
| 🟡 | Orchestrator | Implementera riktig LRU-cache |
| 🟡 | Workers | Förbättra stuck-detektering |
| 🟡 | Server | Validera alla indata |
| 🟡 | Verktyg | Lägg till audit-loggning |

---

## Statistik

| Kategori | Antal |
|----------|-------|
| 🔴 Kritsika buggar | 12 |
| 🟠 Högprioriterade | 8 |
| 🟡 Medelprioriterade | 15 |
| 🟢 Förbättringsförslag | 25 |
| **Totalt** | **60** |

---

*Analys genererad av Navi Workers 2026-04-02*