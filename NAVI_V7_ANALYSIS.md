# Navi-v7 Analys — Server & Själv

**Datum:** 2026-04-06T22:39:00Z  
**Analytiker:** Navi v7 (självanalys)  
**Scope:** Server-kod (/root/navi-v7/), Kunskapsfiler, Självkännedom

---

## 1. Server-analys

### 1.1 Kritiska Buggar

#### BUG #1: Worker — verktygstimeout utan graceful recovery
**Fil:** `worker.js` (rad ~850)  
**Problem:** Verktyg som timeout:ar efter 90s har ingen graceful recovery. Workern fortsätter men verktygsresultatet kan vara ofullständigt.  
**Påverkan:** Kan leda till inkonsekvent state om workern försöker använda ofullständigt resultat.  
**Åtgärd:** Lägg till explicit timeout-handling i verktyget med `Promise.race` och partial-result return.

```javascript
// Exempel på fix:
const TOOL_TIMEOUT_MS = 90_000;
try {
  result = await Promise.race([toolPromise, toolTimeoutPromise]);
} catch (error) {
  if (error.message.includes('timed out')) {
    // Returnera partial result istället för att fortsätta
    return { success: false, error: 'Timeout', partial: true };
  }
}
```

#### BUG #2: Worker — context window check körs efter LLM-anrop
**Fil:** `worker.js` (rad ~380)  
**Problem:** Context-komprimering körs EFTER varje LLM-anrop, inte före. Detta innebär att token-estimatet redan kan ha överskridit gränsen innan varning injiceras.  
**Åtgärd:** Flytta context-check till FÖRE `_callModel` i varje iteration.

#### BUG #3: Worker-pool — dubblett-worker check är för svag
**Fil:** `worker-pool.js` (rad ~380, `findSimilar`)  
**Problem:** `findSimilar()` använder ord-overlap med endast ≥4 gemensamma ord. Detta kan skapa falska positiver för liknande titlar som "Analysera Navi-v7" vs "Analysera Navi-v6".  
**Åtgärd:** Öka tröskeln till ≥6 ord eller använd fuzzy matching.

#### BUG #4: Server — backgroundSyncPushTimestamps växer obegränsat
**Fil:** `server.js` (rad ~280)  
**Problem:** Pruning av `backgroundSyncPushTimestamps` körs var 30:e minut men cutoff är bara 2 timmar. Vid hög aktivitet kan detta växa till tusentals poster.  
**Åtgärd:** Minska cutoff till 30 minuter eller kör pruning var 10:e minut.

#### BUG #5: Memory — learnFromWorkerResult aldrig anropad
**Fil:** `memory.js` (rad ~90)  
**Problem:** Dokumenterad i 28_self_knowledge.md men `learnFromWorkerResult()` anropas aldrig från server.js eller worker-pool.js. Auto-learning fungerar inte.  
**Åtgärd:** Lägg till anrop i `onWorkerDone` callback i server.js.

### 1.2 Förbättringar

#### IMPROVEMENT #1: Orchestrator — ingen deduplicering av identiska meddelanden
**Fil:** `orchestrator.js` (ej lokaliserad rad)  
**Problem:** Om samma meddelande skickas inom kort tid (t.ex. retry efter nätverksfel) kan detta skapa dubbla konversationer.  
**Åtgärd:** Implementera message deduplication baserat på content-hash + timestamp inom 5 sekunder.

#### IMPROVEMENT #2: Worker — ingen progress för plan-uppdateringar
**Fil:** `worker.js`  
**Problem:** När `applyPlanUpdate()` anropas via `message_worker` får användaren ingen progress-notifiering. Detta kan verka som att jobbet har fastnat.  
**Åtgärd:** Lägg till progress-emit i `applyPlanUpdate()`.

#### IMPROVEMENT #3: GitHub-sync — ingen felhantering för enskilda repos
**Fil:** `github-sync.js` (rad ~35)  
**Problem:** Om ett repo failar att klona fortsätter syncen utan felmeddelande för det specifika repot.  
**Åtgärd:** Logga individuella repo-fel och returnera mer detaljerad status.

#### IMPROVEMENT #4: Push-service — ingen token-refresh logic
**Fil:** `push-service.js`  
**Problem:** APNS-tokens kan bli föråldrade men det finns ingen automatisk refresh eller om-registration vid app-uppdatering.  
**Åtgärd:** Lägg till periodisk token-validierung (t.ex. var 24:e timme).

#### IMPROVEMENT #5: Knowledge-loader — cache invalidation är synkron
**Fil:** `knowledge-loader.js` (rad ~45)  
**Problem:** `_isCacheValid()` gör `fs.statSync()` för varje fil vid varje anrop. Detta blockerar vid många filer.  
**Åtgärd:** Gör cache-check asynkron eller använd inotify/watch för filändringsdetektering.

---

## 2. Självanalys (Navi själv)

### 2.1 Dokumenterade begränsningar (från 28_self_knowledge.md)

| Begränsning | Påverkan | Status |
|-------------|----------|--------|
| Kan inte köra Xcode-byggen på servern | iOS-verifiering kräver macOS | Känd |
| `learnFromWorkerResult` inte implementerad | Auto-learning fungerar inte | BUG #5 |
| Deploy-historik överlever restarter | Bra! | Fixad |

### 2.2 Arkitekturproblem

#### PROBLEM #1: Completion gate körs inte alltid
**Fil:** `worker.js` (rad ~450)  
**Problem:** `_runCompletionGate()` körs bara om `[KLAR]` detekteras. Om LLM:n aldrig skickar `[KLAR]` men ändå avslutar (t.ex. pga timeout), körs ingen verifiering.  
**Åtgärd:** Kör completion gate även vid PARTIAL/timeout.

#### PROBLEM #2: Worker watchdog är 90 min men wall-clock är 30 min
**Fil:** `worker.js` (rad ~60)  
**Problem:** Watchdog (90 min) är längre än wall-clock timeout (30 min). Detta innebär att watchdog aldrig triggar före timeout — redundant kod.  
**Åtgärd:** Sänk watchdog till 60 min eller öka wall-clock till 120 min.

#### PROBLEM #3: Ingen rate limiting för orchestrator LLM-anrop
**Fil:** `orchestrator.js`  
**Problem:** Om användaren skickar många meddelanden snabbt kan detta trigga många LLM-anrop utan rate limiting. Kan leda till 429-fel.  
**Åtgärd:** Implementera rate limiting (t.ex. max 10 anrop per minut).

---

## 3. Prioriterad åtgärdslista

| # | Typ | Prioritet | Ämne | Fil |
|---|-----|-----------|------|-----|
| 1 | Bug | KRITISK | learnFromWorkerResult aldrig anropad | memory.js, server.js |
| 2 | Bug | HÖG | Context check efter LLM-anrop | worker.js |
| 3 | Bug | HÖG | Dubblett-worker check för svag | worker-pool.js |
| 4 | Bug | MEDIUM | backgroundSync växer obegränsat | server.js |
| 5 | Bug | MEDIUM | Verktygstimeout utan graceful recovery | worker.js |
| 6 | Förbättring | MEDIUM | Ingen message deduplication | orchestrator.js |
| 7 | Förbättring | MEDIUM | Plan-uppdatering utan progress | worker.js |
| 8 | Förbättring | LÅG | GitHub-sync felhantering | github-sync.js |
| 9 | Förbättring | LÅG | Token refresh logic saknas | push-service.js |
| 10 | Arkitektur | MEDIUM | Completion gate vid timeout | worker.js |

---

## 4. Rekommenderade åtgärder

### Omedelbart (denna session)

1. **Fixa BUG #5** — Anropa `learnFromWorkerResult()` i server.js `onWorkerDone` callback:
   ```javascript
   onWorkerDone: (jobId, result) => {
     // ... existing code ...
     memory.learnFromWorkerResult({
       jobId,
       title: payload.title,
       result: payload.summary,
       filesModified: artifacts.summaryJson?.filesModified,
       learnedAt: new Date().toISOString(),
     });
   }
   ```

2. **Fixa BUG #2** — Flytta context-check före LLM-anrop i worker.js

3. **Fixa PROBLEM #1** — Kör completion gate även vid PARTIAL-status

### Nästa session

4. Förbättra worker-pool findSimilar med fuzzy matching
5. Implementera message deduplication i orchestrator
6. Lägg till token refresh i push-service

---

## 5. Sammanfattning

Analysen har identifierat **10+ buggar och förbättringar** varav **5 är buggar** med varierande allvarlighetsgrad. Den mest kritiska är att auto-learning (`learnFromWorkerResult`) aldrig anropas, vilket betyder att Navi inte lär sig från avslutade jobb.

**Lästa filer:** 12 st
- server.js, worker.js, worker-pool.js, memory.js, push-service.js, github-sync.js, knowledge-loader.js
- 28_self_knowledge.md, 26_agent_architecture.md, 06_memory.md, 05_tools.md, 03_server.md