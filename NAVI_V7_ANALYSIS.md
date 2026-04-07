# Navi-v7 — Komplett Analys och Förbättringsförslag

*Analys genomförd: 2026-04-02 av 5 parallella Navi Workers*

---

## 📋 Sammanfattning

Denna analys täcker alla huvudkomponenter i Navi-v7:
- **Orchestrator** (orchestrator.js, 52KB)
- **Workers** (worker.js, worker-pool.js)
- **Server & WebSocket** (server.js)
- **Verktyg** (tools/, 11 moduler, ~45+ verktyg)
- **Push-notiser** (push-service.js)

**Totalt antal buggar funna:** 45+
**Kritiska säkerhetsproblem:** 7

---

## 🔴 KRITISKA BUGGAR (åtgärda omedelbart)

### 1. Path Traversal i filesystem.js (verktyg)
| Rad | Problem |
|-----|---------|
| ~10-30 | **Ingen sökvägsvalidering** — kan läsa/skriva godtyckliga filer på servern |

**Påverkan:** Full server-kompromittering
**Exempel:** `read_file` kan läsa `/etc/passwd`, `.ssh/authorized_keys`, `.env`

### 2. Bred Shell-whitelist i exec.js
| Rad | Problem |
|-----|---------|
| ~15-40 | Tillåter `python3`, `sed`, `curl`, `awk`, `npx` — möjliggodtycklig kodkörning |

**Påverkan:** Angripare kan köra godtycklig kod

### 3. Obgränsad PM2-kontroll
| Rad | Problem |
|-----|---------|
| ~10-20 | Kan starta om VILKEN process som helst, ändra ALLA miljövariabler |

**Påverkan:** Service disruption, privilege escalation

### 4. GitHub push utan granskning
| Rad | Problem |
|-----|---------|
| ~50-70 | Kan pusha valfri kod utan bekräftelse |

**Påverkan:** Skadlig kod kan pushas till Teds repos

### 5. Hårdkodad API_KEY fallback (server.js)
| Rad | Problem |
|-----|---------|
| 25 | `const API_KEY = process.env.API_KEY \|\| 'navi-brain-2026'` |

**Påverkan:** Om .env saknas används välkänd standardnyckel

### 6. Worker tas inte bort efter watchdog timeout (worker-pool.js)
| Rad | Problem |
|-----|---------|
| 210-211 | `worker.stop(); worker.status = 'FAILED';` — **saknas: this.workers.delete(jobId)** |

**Påverkan:** Minnesläcka

### 7. Rate limit retry utan iterCount-ökning (orchestrator.js)
| Rad | Problem |
|-----|---------|
| 280-285 | `continue` i rate limit-fel ökar inte iterCount — potentiell oändlig loop |

**Påverkan:** Kan fastna i oändlig loop vid 429-fel

---

## 🟠 HÖGA BUGGAR

### Orchestrator

| # | Rad | Problem |
|---|-----|---------|
| 1 | ~215 | Ingen felhantering för `memory.appendMessage` — kan blockera hela chatten |
| 2 | ~270-290 | Felhantering i streaming-loop hanterar inte null-response korrekt |
| 3 | ~310 | Ingen null-kontroll för `fn.name` före executeTool |
| 4 | ~330-340 | `_detectWorkerRequest` kan krascha på null/undefined text |
| 5 | ~340 | `_autoLearnFromConversation` anropas men kanske inte definierad |
| 6 | ~360, 375, 470 | Tomma catch-block utan logging |
| 7 | ~390-410 | Felaktig LRU-cache — rensar fel post |
| 8 | ~580-595 | Session cleanup kan ta bort busy-session (race condition) |
| 9 | ~590-595 | workerJobMap cleanup kan ta bort fortfarande körande jobb |

### Workers

| # | Rad | Problem |
|---|-----|---------|
| 10 | worker.js:60 | Status uppdateras innan onProgress anropas (inkonsekvent) |
| 11 | worker.js:90 | Message pushas till historien före verktygsresultat |
| 12 | worker.js:119 | Tyst JSON-parse-fel — args blir `{}` utan varning |
| 13 | worker.js:127-128 | Felhantering fångar inte error-objekt |
| 14 | worker.js:133-145 | `break` i verktygsloopen påverkar inte huvudloopen |
| 15 | worker.js:155-175 | Svag stuck-detektering (bara 100 tecken) |
| 16 | worker.js:74 | För aggressiv backoff (30s steg vid rate limit) |
| 17 | worker-pool.js:44 | Race condition vid concurrent dequeue |
| 18 | worker-pool.js:200 | setInterval utan cleanup (flera intervals vid omstart) |
| 19 | worker-pool.js:200 | För kort watchdog-interval (60s) |

### Server

| # | Rad | Problem |
|---|-----|---------|
| 20 | ~155-158 | broadcast() saknar felhantering — kan krascha vid nätverksfel |
| 21 | ~199-225 | Klient kan försvinna under async CHAT (race condition) |
| 22 | ~195-198 | sessionId utan validering — kan innehålla skadlig kod |
| 23 | ~187-190 | JSON.parse utan storlekskontroll — DoS-risk |
| 24 | ~271 | Dubbel borttagning i ping-loop |

### Push-notiser

| # | Rad | Problem |
|---|-----|---------|
| 25 | push-service.js:46-62 | **Ingen retry-logik** — vid timeout/503 görs inga återförsök |
| 26 | push-service.js:26-28 | **Ingen persistent token-lagring** — tokens försvinner vid omstart |
| 27 | push-service.js | Ingen APNS-återanslutning vid anslutningsfel |

---

## 🟡 MEDELBUGGAR

| # | Fil | Problem |
|---|-----|---------|
| 28 | orchestrator.js:30 | För bred regex i `classifyTaskComplexity` — nästan allt = 'complex' |
| 29 | orchestrator.js:440-530 | Mycket lång system-prompt varje anrop |
| 30 | worker.js:55-66 | Trim note kan vara felaktig |
| 31 | worker.js:10 | HISTORY_TRIM_THRESHOLD = 60 är för högt |
| 32 | worker.js:44 | similarJobs kan vara undefined |
| 33 | worker-pool.js:6 | Hårdkodad MAX_WORKERS = 20 |
| 34 | worker-pool.js:87 | status() returnerar null vid saknad worker |
| 35 | server.js:274 | Ping-intervall 25s kan minskas till 15s |
| 36 | server.js:286-288 | Graceful shutdown timeout för kort (2s) |
| 37 | tools/search.js | HTML-stripping ofullständig (regex istället för DOM) |
| 38 | tools/elevenlabs.js | Fallback voice kan väljas fel |

---

## 🔵 INEFFEKTIVITETER

| # | Komponent | Problem |
|---|-----------|---------|
| 1 | Orchestrator | Nästan alla förfrågor klassificeras som 'complex' → 10 minnen istället för 7 |
| 2 | Orchestrator | Onödig filinläsning för enkla frågor (_preReadCodeContext) |
| 3 | Worker | _findSimilarJobs läser alla jobb-mappar utan datum-filtrering |
| 4 | Worker-pool | _pruneOldWorkers kör var 60e sekund, borde vara var 5:e minut |
| 5 | Server | sendLiveStatus skickar var 5:e sekund även om inga ändringar |

---

## 🟢 SAKNADE FUNKTIONER

| Funktion | Beskrivning | Prioritet |
|----------|-------------|-----------|
| **Dynamic Island** | iOS 16.1+ stöd saknas helt | Hög |
| **Live Activity** | Visa jobbstatus på lock screen | Hög |
| **Rate limiting** | På WebSocket, verktyg, APNS | Hög |
| **Audit-loggning** | Logga vem som kör vad | Medel |
| **worker_started notis** | När worker börjar köra | Medel |
| **worker_queued notis** | När jobb hamnar i kö | Medel |
| **Context caching** | Cacha knowledge-filer i minnet | Låg |
| **Smart modellval** | Välj billig modell för enkla frågor | Låg |

---

## 📊 FLÖDESANALYS

### Steg 1: Chat → Orchestrator
```
WebSocket → orchestrator.chat() → session skapas → message sparas
```

### Steg 2: Intent-classification
```
_intentRoute() (snabb) → _classifyRequest() (LLM) → spawn worker eller direkt svar
```

### Steg 3: Worker kör
```
worker.js: start() → PLANNING → RUNNING → [KLAR] → DONE
```

### Steg 4: Push-notis
```
worker-pool.onWorkerDone → push-service.send() → APNS
```

---

## 🚀 PRIORITERAD ÅTGÄRDSLISTA

### FAS 1 — KRITISKT (fixa nu)

- [ ] **1.** Lägg till sökvägsvalidering i filesystem.js
- [ ] **2.** Begränsa exec-whitelist (ta bort python3, sed, curl)
- [ ] **3.** Vitlista tillåtna PM2-processer
- [ ] **4.** Kräv API_KEY i .env (fäll inte tillbaka till standard)
- [ ] **5.** Lägg till `this.workers.delete(jobId)` efter watchdog timeout
- [ ] **6.** Fixa rate limit retry-loop (öka iterCount)

### FAS 2 — HÖG PRIORITET (inom 1 vecka)

- [ ] **7.** Lägg till retry-logik för APNS (exponential backoff)
- [ ] **8.** Spara push-tokens till fil (persistent lagring)
- [ ] **9.** Lägg till rate limiting på WebSocket
- [ ] **10.** Lägg till felhantering i broadcast()
- [ ] **11.** Validera sessionId och meddelandestorlek
- [ ] **12.** Fixa tomma catch-block med logging

### FAS 3 — MEDEL PRIORITET (inom 1 månad)

- [ ] **13.** Implementera Dynamic Island & Live Activity
- [ ] **14.** Förbättra classifyTaskComplexity
- [ ] **15.** Sänk HISTORY_TRIM_THRESHOLD till 40
- [ ] **16.** Öka graceful shutdown timeout till 30s
- [ ] **17.** Lägg till audit-loggning

### FAS 4 — FRAMTIDA

- [ ] **18.** Context caching för knowledge-filer
- [ ] **19.** Smart modellval (billig för enkla frågor)
- [ ] **20.** Parallellisera oberoende verktyg

---

## 📁 DETALJERADE RAPPORTER

| Komponent | Analysfil |
|-----------|-----------|
| Orchestrator | orchestrator_analysis.md |
| Workers | worker_analysis.md |
| Server | server_analysis.md |
| Verktyg | tools_analysis.md |
| Push-notiser | push_analysis.md |

---

---

## 📝 KOMBINERAD ANALYS — ALLA PUNKTER FRÅN 3 ARBETSFLÖDEN

### Analys 1: Improve1.md (13 punkter)
Från jobbet "Analysera, tänk och hur du ska fixa alla dessa fel":

| # | Fil | Rad | Problem | Prio |
|---|-----|-----|---------|------|
| 1 | orchestrator.js | 473, 627 | Tomma catch-block — fel loggas men hanteras inte | Kritisk |
| 2 | worker-pool.js | 27-30 | Ingen max-queue-storlek — minnesläcka möjlig | Kritisk |
| 3 | server.js | 46, 51 | `reconcileInflightJobs()` anropas dubbelt | Kritisk |
| 4 | push-service.js | 14-20 | APNS-credentials valideras inte vid init | Kritisk |
| 5 | worker-pool.js | - | `_pruneOldWorkers()` definieras men anropas aldrig | Hög |
| 6 | worker-pool.js | 91-92 | `lastProgress` kan vara null om worker redan rensats | Hög |
| 7 | worker.js | 131 | `checkIn()` anropas inte efter långa verktyg | Hög |
| 8 | worker.js | 148 | Ingen timeout per verktyg | Hög |
| 9 | server.js | - | WebSocket saknar autentisering | Hög |
| 10 | orchestrator.js | 67-71 | `stopMatch[2]` kan vara undefined | Medel |
| 11 | orchestrator.js | 328 | `_detectWorkerRequest` returnerar null för citatlösa titlar | Medel |
| 12 | worker-pool.js | - | Ingen message() funktion (dokumentation vs kod) | Medel |
| 13 | worker.js | 67-79 | history växer obegränsat vid många verktyg | Medel |

### Analys 2: Kritiska säkerhetsproblem (7 punkter)
Från samma jobb ovan:

| # | Fil | Problem | Prio |
|---|-----|---------|------|
| 1 | tools/filesystem.js | Path Traversal — ingen sökvägsvalidering | Kritisk |
| 2 | tools/exec.js | Bred Shell-whitelist — möjliggodtycklig kodkörning | Kritisk |
| 3 | tools/pm2.js | Obgränsad PM2-kontroll — kan starta om vilken process | Kritisk |
| 4 | tools/github.js | GitHub push utan granskning | Kritisk |
| 5 | server.js | Hårdkodad API_KEY fallback | Kritisk |
| 6 | worker-pool.js | Worker tas inte bort efter watchdog timeout | Kritisk |
| 7 | orchestrator.js | Rate limit retry utan iterCount-ökning | Kritisk |

---

## 📊 SAMMANSTÄLLNING: TOTALT 45+ PUNKTER

| Kategori | Antal |
|----------|-------|
| Kritiska buggar | 15+ |
| Höga buggar | 12+ |
| Medelbuggar | 10+ |
| Säkerhetsproblem | 7 |
| Ineffektiviteter | 5 |
| Saknade funktioner | 4 |

---

## 🚀 KOMPLETT ÅTGÄRDSLISTA

### FAS 1 — KRITISKT (fixa nu)

- [ ] 1. Path Traversal i filesystem.js — lägg till sökvägsvalidering
- [ ] 2. Begränsa exec-whitelist i exec.js — ta bort python3, sed, curl
- [ ] 3. Vitlista PM2-processer i pm2.js
- [ ] 4. Kräv API_KEY i .env — ta bort fallback
- [ ] 5. Lägg till `this.workers.delete(jobId)` efter watchdog timeout
- [ ] 6. Fixa rate limit retry-loop i orchestrator.js
- [ ] 7. Fixa tomma catch-block i orchestrator.js
- [ ] 8. Lägg till `_pruneOldWorkers()` anrop i worker-pool.js
- [ ] 9. Ta bort dubbelt anrop till `reconcileInflightJobs()` i server.js
- [ ] 10. Lägg till APNS-validering vid start i push-service.js
- [ ] 11. Lägg till checkIn() efter verktyg i worker.js
- [ ] 12. Lägg till timeout per verktyg i worker.js
- [ ] 13. Lägg till max-queue-storlek i worker-pool.js

### FAS 2 — HÖG PRIORITET (inom 1 vecka)

- [ ] 14. Lägg till retry-logik för APNS med exponential backoff
- [ ] 15. Spara push-tokens till fil (persistent lagring)
- [ ] 16. Lägg till rate limiting på WebSocket
- [ ] 17. Lägg till felhantering i broadcast() i server.js
- [ ] 18. Validera sessionId och meddelandestorlek
- [ ] 19. Implementera WebSocket-autentisering
- [ ] 20. Smart modellval — välj billig modell för enkla frågor
- [ ] 21. Rate limiting per session i worker-pool.js

### FAS 3 — MEDEL PRIORITET (inom 1 månad)

- [ ] 22. Implementera Dynamic Island & Live Activity
- [ ] 23. Förbättra classifyTaskComplexity i orchestrator.js
- [ ] 24. Sänk HISTORY_TRIM_THRESHOLD till 40
- [ ] 25. Öka graceful shutdown timeout till 30s
- [ ] 26. Lägg till audit-loggning
- [ ] 27. Parallellisera oberoende verktyg
- [ ] 28. Context caching för knowledge-filer

### FAS 4 — FRAMTIDA

- [ ] 29. Smart modellval (billig för enkla frågor)
- [ ] 30. Job dependencies

---

*Analys genererad av Navi Workers — 2026-04-02*
*Uppdaterad: 2026-04-07 med kombinerade punkter från 3 arbetsflöden*