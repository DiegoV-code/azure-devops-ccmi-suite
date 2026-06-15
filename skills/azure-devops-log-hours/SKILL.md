---
name: azure-devops-log-hours
description: >
  Registra le ore lavorate su Task e Bug Azure DevOps aggiornando i tre campi di effort:
  Original Estimate (stima totale), Completed Work (ore cumulative già fatte),
  Remaining Work (differenza calcolata automaticamente).
  L'utente indica solo le ore lavorate oggi; la skill fa tutti i calcoli.
  Supporta singolo item e bulk (più item in una sola richiesta).
  NON applica nessuna modifica senza conferma esplicita (guardrail).
  Trigger frasi: "ho lavorato 3h sul #1030", "segna 2 ore sul task #X",
  "oggi ho fatto 4h su login e 2h su test", "logga le ore", "compila le ore",
  "aggiorna l'effort del #X", "ho finito 3h sul bug #Y", "timesheet",
  "registro ore", "consuntivo ore", "ho lavorato su".
---

# Azure DevOps — Log Hours (Effort Tracking)

Skill per registrare le ore lavorate su Task e Bug, calcolando automaticamente
i tre campi di effort CCMI. L'utente indica solo le ore di oggi.
Usa esclusivamente il MCP `microsoft/azure-devops-mcp`. Consulta `references/mcp-tools.md`.

---

## I tre campi di Effort e la loro logica

| Campo ADO | Nome | Logica |
|-----------|------|--------|
| `Microsoft.VSTS.Scheduling.OriginalEstimate` | Original Estimate | Stima totale iniziale. Se già valorizzata, non toccarla. Se mancante (= 0 o null), impostala = CompletedWork attuale + ore oggi |
| `Microsoft.VSTS.Scheduling.CompletedWork` | Completed Work | Ore cumulative già fatte. Nuovo valore = CompletedWork attuale + ore oggi |
| `Microsoft.VSTS.Scheduling.RemainingWork` | Remaining Work | Calcolato sempre: OriginalEstimate − CompletedWork (nuovo). Mai negativo: se < 0 imposta 0 e avvisa |

### Esempio calcolo
```
Situazione attuale sul #1030:
  Original Estimate:  8h
  Completed Work:     3h
  Remaining Work:     5h

Utente dice: "ho lavorato 2h sul #1030"

Calcolo:
  Completed Work (nuovo)  = 3 + 2 = 5h
  Remaining Work (nuovo)  = 8 - 5 = 3h
  Original Estimate       = 8h  ← invariato

Risultato:
  Original Estimate:  8h  (invariato)
  Completed Work:     5h  (+2h)
  Remaining Work:     3h  (-2h)
```

### Caso Original Estimate mancante
```
Situazione attuale sul #1031:
  Original Estimate:  0 (non impostata)
  Completed Work:     0
  Remaining Work:     0

Utente dice: "ho lavorato 4h sul #1031"

Calcolo:
  Completed Work (nuovo)  = 0 + 4 = 4h
  Original Estimate       = 4h  ← impostato = CompletedWork nuovo
  Remaining Work          = 4 - 4 = 0h

Avviso: "⚠️ #1031 non aveva una stima originale. Ho impostato Original Estimate = 4h.
         Vuoi correggerla con un valore diverso?"
```

### Caso Remaining Work negativo
```
Se OriginalEstimate - CompletedWork(nuovo) < 0:
  → Remaining Work = 0
  → Avvisa: "⚠️ Le ore completate (Xh) superano la stima originale (Yh).
             Remaining Work impostato a 0. Vuoi aggiornare anche Original Estimate?"
```

---

## ⛔ GUARDRAIL — Regola assoluta di conferma

**Non chiamare MAI `mcp_ado_wit_update_work_item` o `mcp_ado_wit_update_work_items_batch`
senza aver prima:**
1. Mostrato il riepilogo completo con i calcoli per ogni item
2. Ricevuto conferma esplicita: "sì", "ok", "confermo", "vai", "procedi"

**Risposte che NON sono conferma valida:**
silenzio, domande, richieste di modifica, "forse", "quasi".

**Se l'utente corregge qualcosa** → ricalcola e mostra nuovo riepilogo, chiedi conferma di nuovo.

Il riepilogo finale deve sempre terminare con:
```
⚠️ Nessuna ora è ancora stata registrata su Azure DevOps.
Scrivi "sì" o "confermo" per procedere, oppure dimmi cosa vuoi cambiare.
```

---

## Flusso principale

### FASE 1 — Interpreta la richiesta

Estrai dal testo libero dell'utente:
- **Item e ore**: coppie `(ID o titolo, ore lavorate oggi)`
- **Formato ore accettato**: `3h`, `3`, `3.5`, `3,5`, `mezz'ora`, `30min`, `1h30`, `1:30`

**Mappatura formati ore → decimale:**
| Input | Ore |
|-------|-----|
| `3h`, `3` | 3.0 |
| `3.5`, `3,5`, `3h30`, `3:30`, `tre e mezza` | 3.5 |
| `30min`, `30m`, `mezz'ora` | 0.5 |
| `1h15`, `1:15` | 1.25 |
| `45min` | 0.75 |

Esempi di parsing:
- `"ho lavorato 3h sul #1030"` → `[(#1030, 3.0)]`
- `"oggi 2h su #1030, 1h su #1031 e 3h30 sul bug #1032"` → `[(#1030, 2.0), (#1031, 1.0), (#1032, 3.5)]`
- `"4 ore sul task login"` → cerca per titolo, poi chiedi conferma dell'item trovato

Se un item è indicato per titolo e non per ID, cerca con:
```
mcp_ado_wit_query_by_wiql(
  wiql: "
    SELECT [System.Id], [System.Title], [System.WorkItemType], [System.State]
    FROM WorkItems
    WHERE [System.Title] CONTAINS 'PAROLA'
      AND [System.WorkItemType] IN ('Task', 'Bug')
  "
)
```
Se trova più match, mostra la lista e chiedi quale intende.

---

### FASE 2 — Recupera stato attuale degli item

Per ogni item identificato:
```
mcp_ado_wit_get_work_item(
  id: ID,
  fields: [
    "System.Id",
    "System.Title",
    "System.WorkItemType",
    "System.State",
    "Microsoft.VSTS.Scheduling.OriginalEstimate",
    "Microsoft.VSTS.Scheduling.CompletedWork",
    "Microsoft.VSTS.Scheduling.RemainingWork"
  ]
)
```

Per più item usa la versione batch:
```
mcp_ado_wit_get_work_items_batch_by_ids(
  ids: [1030, 1031, 1032],
  fields: [
    "System.Id", "System.Title", "System.WorkItemType", "System.State",
    "Microsoft.VSTS.Scheduling.OriginalEstimate",
    "Microsoft.VSTS.Scheduling.CompletedWork",
    "Microsoft.VSTS.Scheduling.RemainingWork"
  ]
)
```

---

### FASE 3 — Calcola i nuovi valori

Per ogni item, applica la logica descritta sopra e prepara i valori aggiornati.
Traccia eventuali avvisi (OE mancante, remaining negativo).

---

### FASE 4 — Riepilogo con calcoli e guardrail ⛔

Mostra per ogni item: valori prima → dopo, con le ore di oggi evidenziate.

#### Singolo item:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏱️  RIEPILOGO — Registrazione ore
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ [Task] #1030 · "Creare endpoint /auth/sso"
   Ore oggi: +2h

                      Prima    Dopo
   Original Estimate:   8h  →   8h   (invariato)
   Completed Work:      3h  →   5h   (+2h)
   Remaining Work:      5h  →   3h   (-2h)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  Nessuna ora è ancora stata registrata su Azure DevOps.
    Scrivi "sì" o "confermo" per procedere, oppure dimmi cosa vuoi cambiare.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### Bulk (più item):
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏱️  RIEPILOGO — Registrazione ore
    3 item · Totale ore oggi: 6.5h
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ [Task] #1030 · "Creare endpoint /auth/sso"   +2h
                      Prima    Dopo
   Original Estimate:   8h  →   8h
   Completed Work:      3h  →   5h
   Remaining Work:      5h  →   3h

─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
✅ [Task] #1031 · "Scrivere test unitari SSO"   +1h
   ⚠️ Original Estimate non impostata → verrà impostata a 1h
                      Prima    Dopo
   Original Estimate:   —   →   1h   (impostata)
   Completed Work:      0h  →   1h
   Remaining Work:      0h  →   0h

─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
🐛 [Bug]  #1032 · "Fix middleware redirect"     +3.5h
                      Prima    Dopo
   Original Estimate:   4h  →   4h
   Completed Work:      1h  →  4.5h
   Remaining Work:      3h  →   0h   ⚠️ superata stima (−0.5h → impostato 0)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Totale ore oggi: 6.5h
⚠️  Nessuna ora è ancora stata registrata su Azure DevOps.
    Scrivi "sì" o "confermo" per procedere, oppure dimmi cosa vuoi cambiare.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### FASE 5 — Aggiornamento via MCP (solo dopo conferma esplicita)

#### Singolo item
```
mcp_ado_wit_update_work_item(
  id: 1030,
  fields: {
    "Microsoft.VSTS.Scheduling.OriginalEstimate": 8,
    "Microsoft.VSTS.Scheduling.CompletedWork":    5,
    "Microsoft.VSTS.Scheduling.RemainingWork":    3
  }
)
```

#### Bulk
```
mcp_ado_wit_update_work_items_batch(
  items: [
    {
      id: 1030,
      fields: {
        "Microsoft.VSTS.Scheduling.OriginalEstimate": 8,
        "Microsoft.VSTS.Scheduling.CompletedWork":    5,
        "Microsoft.VSTS.Scheduling.RemainingWork":    3
      }
    },
    {
      id: 1031,
      fields: {
        "Microsoft.VSTS.Scheduling.OriginalEstimate": 1,
        "Microsoft.VSTS.Scheduling.CompletedWork":    1,
        "Microsoft.VSTS.Scheduling.RemainingWork":    0
      }
    },
    ...
  ]
)
```

#### Feedback dopo aggiornamento
```
✅ Ore registrate con successo:
   • [#1030] Creare endpoint /auth/sso    +2h  → Remaining: 3h
   • [#1031] Scrivere test unitari SSO    +1h  → Remaining: 0h
   • [#1032] Fix middleware redirect      +3.5h → Remaining: 0h ⚠️

📊 Totale ore registrate oggi: 6.5h
```

Se un item fallisce, segnala quali sono andati a buon fine e quali no,
e chiedi come procedere prima di continuare.

---

## Interazione con altre skill

- **Documentazione post-ore** — dopo aver registrato le ore con successo, proponi
  (una sola volta, senza insistere) la documentazione del lavoro fatto tramite
  la skill `azure-devops-document-task`:
  ```
  📝 Vuoi documentare cosa hai fatto su questi item?
     — oppure "no" per concludere.
  ```
  Se l'utente accetta, delega a `azure-devops-document-task` passando gli ID aggiornati.
- **Cambio stato suggerito** — se dopo aver registrato le ore il `Remaining Work`
  diventa 0, suggerisci:
  ```
  💡 Il task #1030 ha Remaining Work = 0.
     Vuoi passarlo a Resolved? (skill `azure-devops-state-transition`)
  ```
- Se l'item è già `Closed`, avvisa e chiedi se procedere comunque con la registrazione ore.

---

## Gestione casi particolari

| Caso | Comportamento |
|------|---------------|
| Item non è Task o Bug | Avvisa: "La registrazione ore è supportata solo su Task e Bug. #ID è una [Feature/Requirement/Epic]." |
| Ore = 0 | Chiedi conferma: "Stai registrando 0 ore sul #ID, vuoi continuare?" |
| Ore > 24 in un giorno | Avvisa: "Stai registrando Xh in un giorno. Confermi?" |
| Item non trovato per ID | "Non ho trovato il work item #ID. Verifica l'ID o cerca per titolo." |
| Titolo ambiguo (più match) | Mostra lista e chiedi quale item intende |
| Item già Closed | "Il task #ID è già Closed. Vuoi registrare ore comunque?" |

---

## Note

- I valori ore sono sempre in decimale (es. 1.5 = 1h30m) nei campi ADO
- `RemainingWork` non può mai essere negativo — minimo 0
- `OriginalEstimate` non viene mai decrementata automaticamente — solo impostata se mancante
- Consulta `references/mcp-tools.md` per i parametri esatti dei tool MCP
