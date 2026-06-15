---
name: azure-devops-state-transition
description: >
  Cambia lo stato (Proposed, Active, Resolved, Closed, Blocked) di work item Azure DevOps
  secondo la gerarchia CCMI: Epic → Feature → Requirement/Bug → Task.
  Supporta singolo item, bulk e cascata su tutta la gerarchia figlio.
  NON applica nessuna modifica senza conferma esplicita dell'utente (guardrail).
  Trigger frasi: "chiudi il task #X", "metti Active il requirement #Y",
  "segna come resolved la feature Z", "chiudi tutti i task del requirement #X",
  "passa a closed l'epic #Y con tutti i figli", "aggiorna lo stato di",
  "mark as done", "completa il task", "riapri il #X", "blocca il #X".
---

# Azure DevOps — State Transition

Skill per cambiare lo stato dei work item Azure DevOps rispettando la gerarchia CCMI.
Supporta singolo item, bulk e cascata su figli.
Usa il MCP `microsoft/azure-devops-mcp`. Consulta `references/mcp-tools.md`.

---

## ⛔ GUARDRAIL — Regola assoluta di conferma

**Non chiamare MAI `mcp_ado_wit_update_work_item` o `mcp_ado_wit_update_work_items_batch`
senza aver prima:**
1. Mostrato all'utente il riepilogo completo di tutti gli item che verranno modificati
2. Ricevuto una conferma esplicita: "sì", "ok", "confermo", "vai", "procedi"

**Vale anche per le cascate:** se il cambio stato coinvolge 1 item o 50 item figli,
il riepilogo mostra TUTTI gli item che cambieranno, uno per uno.

**Risposte che NON sono conferma valida:**
- Silenzio, domande, richieste di modifica, "forse", "quasi"

**Se l'utente modifica qualcosa dopo il riepilogo** → aggiorna il riepilogo e chiedi
conferma di nuovo da zero.

Il riepilogo finale deve sempre terminare con:
```
⚠️ Nessuno stato è ancora stato modificato su Azure DevOps.
Scrivi "sì" o "confermo" per procedere, oppure dimmi cosa vuoi cambiare.
```

---

## Stati CCMI e transizioni valide

```
Proposed ──→ Active ──→ Resolved ──→ Closed
    ↑            │                      │
    │            ↓                      │
    └────── Blocked ←───────────────────┘
                 │
                 └──→ Active (sblocco)
```

### Transizioni permesse per tipo

| Da → A | Proposed | Active | Blocked | Resolved | Closed |
|--------|----------|--------|---------|----------|--------|
| Proposed | — | ✅ | ✅ | ❌ | ❌ |
| Active | ✅ | — | ✅ | ✅ | ✅ |
| Blocked | ✅ | ✅ | — | ✅ | ❌ |
| Resolved | ❌ | ✅ | ❌ | — | ✅ |
| Closed | ❌ | ✅ | ❌ | ✅ | — |

Se l'utente richiede una transizione non valida (es. Proposed → Resolved),
avvisa e proponi il percorso corretto:
```
⚠️ Non puoi passare direttamente da Proposed a Resolved.
   Percorso suggerito: Proposed → Active → Resolved
   Vuoi che aggiorni entrambi gli stati in sequenza?
```

---

## Flusso principale

### FASE 1 — Interpreta la richiesta

Identifica dalla richiesta:
- **Target**: ID specifico, titolo, tipo + filtro (es. "tutti i task del #1020"), bulk
- **Nuovo stato**: esplicito ("chiudi", "metti Active") o da proporre (se non specificato)
- **Cascata**: se il target è Epic/Feature, prepara la cascata su tutti i figli

**Mappatura linguaggio naturale → stato:**
| Parole chiave | Stato |
|--------------|-------|
| "chiudi", "done", "completa", "finito" | Closed |
| "attiva", "inizia", "in corso", "parti" | Active |
| "risolvi", "resolved", "risolto" | Resolved |
| "proponi", "rimetti in backlog", "riapri" | Proposed |
| "blocca", "bloccato", "blocked" | Blocked |

---

### FASE 2 — Recupera item coinvolti

#### Singolo item per ID
```
mcp_ado_wit_get_work_item(
  id: ID,
  fields: ["System.Id", "System.Title", "System.State",
           "System.WorkItemType", "System.Parent"]
)
```

#### Bulk — tutti i figli di un item (cascata)
```
mcp_ado_wit_query_by_wiql(
  wiql: "
    SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType]
    FROM WorkItemLinks
    WHERE [Source].[System.Id] = PARENT_ID
      AND [System.Links.LinkType] = 'System.LinkTypes.Hierarchy-Forward'
    MODE (Recursive)
  "
)
```

#### Bulk — per tipo e filtro
```
mcp_ado_wit_query_by_wiql(
  wiql: "
    SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType]
    FROM WorkItems
    WHERE [System.Parent] = PARENT_ID
      AND [System.WorkItemType] = 'Task'
      AND [System.State] <> 'Closed'
  "
)
```

---

### FASE 3 — Proposta stato (se non specificato)

Se l'utente non ha indicato lo stato target, proponi le opzioni valide
in base allo stato attuale dell'item:

```
📄 [Requirement] #1020 · "Implementare login con SSO"
   Stato attuale: 🟢 Active

   In quale stato vuoi portarlo?
   1. 🟣 Resolved   — lavoro completato, in attesa di verifica
   2. ⚫ Closed     — chiuso definitivamente
   3. 🔴 Blocked    — bloccato da impedimento
   4. 🔵 Proposed   — rimetti in backlog
```

---

### FASE 4 — Calcola impatto cascata

Se il target è **Epic o Feature** o l'utente chiede una modifica bulk,
recupera TUTTI i figli ricorsivamente e mostra l'impatto completo prima del riepilogo.

**Ordine di applicazione della cascata (bottom-up):**
```
Task → Requirement/Bug → Feature → Epic
```
I figli vanno aggiornati prima dei padri per rispettare le regole ADO.

**Avviso se la cascata è grande:**
```
⚠️ Questa operazione coinvolge 23 item. Vuoi continuare?
```
Se > 20 item, chiedi conferma esplicita solo sulla dimensione prima di mostrare il riepilogo completo.

---

### FASE 5 — Riepilogo finale con guardrail ⛔

Mostra tutti gli item che verranno modificati, raggruppati per tipo nella gerarchia.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔄 RIEPILOGO — Cambio di stato
   Progetto: [PROGETTO]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 Epic
  #1001 · Gestione utenti          Active → Closed

  📋 Feature
    #1010 · Login e autenticazione   Active → Closed

    📄 Requirement
      #1020 · Login con SSO          Active → Closed
        ✅ Task
          #1030 · Endpoint /auth/sso   Active  → Closed
          #1031 · Test unitari SSO     Proposed → Closed

      🐛 Bug
        #1021 · Redirect loop login    Blocked → Closed
          ✅ Task
            #1032 · Fix middleware       Blocked → Closed

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Totale: 7 item modificati
   Active → Closed: 4   |   Proposed → Closed: 1   |   Blocked → Closed: 2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  Nessuno stato è ancora stato modificato su Azure DevOps.
    Scrivi "sì" o "confermo" per procedere, oppure dimmi cosa vuoi cambiare.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### FASE 6 — Aggiornamento via MCP (solo dopo conferma esplicita)

#### Singolo item
```
mcp_ado_wit_update_work_item(
  id:     ID,
  fields: { "System.State": "NUOVO_STATO" }
)
```

#### Bulk (più item in una sola chiamata)
```
mcp_ado_wit_update_work_items_batch(
  items: [
    { id: 1030, fields: { "System.State": "Closed" } },
    { id: 1031, fields: { "System.State": "Closed" } },
    { id: 1032, fields: { "System.State": "Closed" } },
    ...
  ]
)
```

**Strategia di esecuzione per le cascate (bottom-up):**
1. Prima aggiorna tutti i **Task**
2. Poi i **Requirement / Bug**
3. Poi le **Feature**
4. Infine le **Epic**

Questo ordine previene errori di validazione ADO dove il padre
non può essere chiuso se ha figli ancora aperti.

#### Feedback dopo l'aggiornamento
```
✅ Aggiornati con successo: 7 item
   • [#1030] Endpoint /auth/sso     → Closed
   • [#1031] Test unitari SSO       → Closed
   • [#1032] Fix middleware         → Closed
   • [#1020] Login con SSO         → Closed
   • [#1021] Redirect loop login   → Closed
   • [#1010] Login e autenticazione → Closed
   • [#1001] Gestione utenti        → Closed
```

Se uno o più item falliscono, mostra quali sono andati a buon fine e quali no,
e chiedi come procedere prima di continuare.

---

## Gestione casi particolari

| Caso | Comportamento |
|------|---------------|
| Transizione non valida | Avvisa, mostra percorso alternativo valido, chiedi conferma |
| Item già nello stato richiesto | Escludi dal riepilogo, notifica: `ℹ️ #ID è già Closed, skip` |
| Item figlio in stato incompatibile con la cascata | Segnala nel riepilogo con `⚠️` e chiedi se forzare o saltare |
| Epic/Feature con figli in sprint attivo | Avvisa: `⚠️ La Feature #X ha task nello sprint corrente. Sicuro di voler chiudere?` |
| ID non trovato | "Non ho trovato il work item #ID. Vuoi cercare per titolo?" |
| Bulk > 20 item | Chiedi conferma sulla dimensione prima del riepilogo completo |

---

## Note

- Usa `mcp_ado_wit_update_work_items_batch` per le cascate — più efficiente di chiamate singole
- L'ordine bottom-up (Task → Req → Feature → Epic) è obbligatorio per ADO
- Dopo ogni aggiornamento riuscito, offri di tornare alla vista gerarchia con `azure-devops-hierarchy-reader`
- Consulta `references/mcp-tools.md` per i parametri esatti dei tool MCP
