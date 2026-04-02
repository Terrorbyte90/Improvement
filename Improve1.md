# Navi-v7 — Analys och Förbättringsförslag

*Sammanställd: 2026-04-02*

---

## 📋 Sammanfattning

Denna fil innehåller en komplett analys av Navi-v7, inklusive buggar, förbättringsförslag och saknade funktioner. Analysen är baserad på djupgående granskning av server-koden.

---

## 🔴 Buggar (15 st)

### Kritiska

| # | Fil | Rad | Problem |
|---|-----|-----|---------|
| 1 | orchestrator.js | 473, 627 | Tomma catch-block — fel loggas men hanteras inte |
| 2 | worker-pool.js | 27-30 | Ingen max-queue-storlek — minnesläcka möjlig |
| 3 | server.js | 46, 51 | `reconcileInflightJobs()` anropas dubbelt |
| 4 | push-service.js | 14-20 | APNS-credentials valideras inte vid init |

### Hög

| # | Fil | Rad | Problem |
|---|-----|-----|---------|
| 5 | worker-pool.js | - | `_pruneOldWorkers()` definieras men anropas aldrig |
| 6 | worker-pool.js | 91-92 | `lastProgress` kan vara null om worker redan rensats |
| 7 | worker.js | 131 | `checkIn()` anropas inte efter långa verktyg |
| 8 | worker.js | 148 | Ingen timeout per verktyg |
| 9 | server.js | - | WebSocket saknar autentisering |

### Medel

| # | Fil | Rad | Problem |
|---|-----|-----|---------|
| 10 | orchestrator.js | 67-71 | `stopMatch[2]` kan vara undefined |
| 11 | orchestrator.js | 328 | `_detectWorkerRequest` returnerar null för citatlösa titlar |
| 12 | worker-pool.js | - | Ingen message() funktion (dokumentation vs kod) |
| 13 | worker.js | 67-79 | history växer obegränsat vid många verktyg |
| 14 | push-service.js | 44-60 | Ingen retry vid APNS-fel |
| 15 | worker.js | 120 | Sekventiell verktygskörning — inge parallellisering |

---

## 🟡 Förbättringar (20+ st)

### Hög prioritet

| # | Fil | Förslag |
|---|-----|---------|
| 1 | worker.js | Smart modellval — välj billig modell för enkla frågor |
| 2 | worker-pool.js | Rate limiting per session |
| 3 | orchestrator.js | Förbättra intent-routning — fler engelska varianter |
| 4 | orchestrator.js | Session cleanup — garanti att gamla sessioner rensas |
| 5 | worker.js | Cacha ofta använda filer (knowledge-filer) |
| 6 | worker-pool.js | Timeout per jobb (konfigurerbar) |
| 7 | push-service.js | Retry-logik för misslyckade notiser |

### Medel prioritet

| # | Fil | Förslag |
|---|-----|---------|
| 8 | worker.js | Parallellisera oberoende verktyg |
| 9 | orchestrator.js | Lägg till intents: `fortsätt`, `pausa`, `restart` |
| 10 | orchestrator.js | Lägg till `visa` som action verb |
| 11 | worker.js | Förbättrad watchdog med uppgiftsspecifika timeouts |
| 12 | tools/index.js | `patch_file` fungerar inte på filer >1MB |
| 13 | orchestrator.js | Max-historia per session — mer konsekvent |
| 14 | server.js | Health check för push-service |

### Låg prioritet

| # | Fil | Förslag |
|---|-----|---------|
| 15 | worker-pool.js | Push-notis när jobb hamnar i kö |
| 16 | push-service.js | Kategorisering av notiser (job_queued, etc.) |
| 17 | worker.js | JSON-strukturad progress istället för fri text |

---

## 🟢 Nya funktioner (4 st)

| # | Funktion | Beskrivning |
|---|----------|-------------|
| 1 | **Context caching** | Cacha knowledge-filer i minnet för snabbare åtkomst |
| 2 | **Smart modellval** | Välj billig modell för status-frågor, dyr för kodning |
| 3 | **Job dependencies** | Låt jobb vänta på andra jobb innan start |
| 4 | **Dynamic Island & Live Activity** | **Saknas helt — behöver implementeras** |

---

## 📊 Flödesanalys

### Steg 1: Chat → Orchestrator
- WebSocket tar emot meddelande → `orchestrator.chat()`
- Session skapas/uppdateras
- Meddelande sparas till history

### Steg 2: Intent-classification
- Fast routing för: status, jobblista, stoppa, hälsning, historik
- LLM för komplexa frågor och worker-spawns
- Problem: vissa svenska/engelska varianter saknas

### Steg 3: Worker spawn & kör
- FIFO-kö med max 20 workers
- Verktyg körs sekventiellt (ej parallellt)
- Progress rapporteras via callbacks

### Steg 4: Job completion & push
- Worker markeras DONE → `onComplete` callback
- Push-notis skickas via APNS
- Problem: inga retries vid APNS-fel

---

## 📱 Notiser, Dynamic Island & Live Activity

### Nuvarande status

| Funktion | Status | Kommentar |
|----------|--------|-----------|
| Push-notiser (APNS) | ✅ Fungerar | Konfigurerad, men inga retries |
| Dynamic Island | ❌ Saknas | Inte implementerat |
| Live Activity | ❌ Saknas | Inte implementerat |

### Problem identifierade

1. **Ingen Dynamic Island** — iOS 16.1+ stödjer detta
2. **Ingen Live Activity** — kan visa jobbstatus på lock screen
3. **APNS timeout** — inget sätt att konfigurera per jobb
4. **Token-registration** — inga retries om det misslyckas

### Förbättringsförslag

1. Implementera ActivityKit för Dynamic Island
2. Lägg till Live Activity för jobb-progress
3. Förbättra APNS-felhantering med retries
4. Lägg till health check för push-service

---

## 🚀 Prioriterad åtgärdslista

### Fas 1 — Kritisk (fixa nu)

1. [ ] Fixa tomma catch-block i orchestrator.js
2. [ ] Lägg till `_pruneOldWorkers()` anrop i worker-pool.js
3. [ ] Ta bort dubbelt anrop till `reconcileInflightJobs()`
4. [ ] Lägg till APNS-validering vid start
5. [ ] Lägg till checkIn() efter verktyg i worker.js

### Fas 2 — Hög prioritet

6. [ ] Implementera smart modellval
7. [ ] Lägg till rate limiting
8. [ ] Förbättra intent-routning
9. [ ] Implementera timeout per jobb
10. [ ] Lägg till retry för APNS

### Fas 3 — Framtida

11. [ ] Dynamic Island & Live Activity
12. [ ] Context caching
13. [ ] Job dependencies
14. [ ] Parallellisera verktyg

---

*Analys genomförd av Navi Workers — 2026-04-02*