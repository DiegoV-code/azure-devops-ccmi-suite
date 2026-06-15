---
name: azure-devops-hierarchy-reader
description: >
  Skill centrale per leggere e navigare la gerarchia completa dei work item Azure DevOps
  secondo il processo CCMI: Epic → Feature → Requirement/Bug → Task.
  Mostra l'albero a qualsiasi livello di profondità in base a ciò che chiede l'utente.
  Può richiamare la skill azure-devops-daily-tasks per creare nuovi item all'interno
  della gerarchia visualizzata.
  Trigger frasi: "mostrami la gerarchia", "dimmi cosa c'è sotto questa epic",
  "mostrami la feature X", "albero dei work item", "struttura del backlog",
  "cosa c'è sotto il requirement #123", "esplora l'epic Y", "mostrami tutto",
  "apri il backlog", "leggi gli item", "struttura CCMI", "espandi la feature",
  "cosa contiene l'epic", "naviga il backlog".
---

# Azure DevOps — CCMI Hierarchy Reader

Skill centrale di **lettura e navigazione** della gerarchia work item Azure DevOps.
Processo: **CCMI** → `Epic → Feature → Requirement | Bug → Task`
Può delegare la creazione alla skill `azure-devops-daily-tasks`.
Usa esclusivamente tool MCP di lettura. Consulta `references/mcp-tools.md`.

---

## Gerarchia CCMI

```
Epic                          (Livello 1 — massimo)
└── Feature                   (Livello 2)
    ├── Requirement            (Livello 3)
    │   └── Task               (Livello 4 — minimo)
    └── Bug                    (Livello 3, alternativo a Requirement)
        └── Task               (Livello 4)
```

**Regole della gerarchia:**
- Un `Task` appartiene sempre a un `Requirement` o a un `Bug`
- Un `Requirement` o `Bug` appartiene sempre a una `Feature`
- Una `Feature` appartiene sempre a una `Epic`
- I `Task` non hanno figli

---

## Formato di visualizzazione

Ogni nodo mostra: **stato** + **ID** + **titolo**. Niente altro di default.

### Legenda stati CCMI
```
🔵 Proposed   🟢 Active   🔴 Blocked   🟣 Resolved   ⚫ Closed
```

### Esempio albero completo
```
📦 Epic
└── 🟢 [Active]   #1001 · Gestione utenti

    📋 Feature
    ├── 🟢 [Active]   #1010 · Login e autenticazione
    │
    │   📄 Requirement / 🐛 Bug
    │   ├── 🟢 [Active]   #1020 · Implementare login con SSO
    │   │   └── ✅ Task
    │   │       ├── 🟢 [Active]   #1030 · Creare endpoint /auth/sso
    │   │       └── 🔵 [Proposed] #1031 · Scrivere test unitari SSO
    │   │
    │   └── 🔴 [Blocked]  #1021 · Bug: redirect loop dopo login
    │       └── ✅ Task
    │           └── 🔴 [Blocked]  #1032 · Fix middleware redirect
    │
    └── 🔵 [Proposed] #1011 · Gestione profilo utente
        └── 📄 Requirement
            └── 🔵 [Proposed] #1022 · Form modifica profilo
                └── ✅ Task
                    └── 🔵 [Proposed] #1033 · UI form profilo
```

---

## Interpretazione della richiesta

La skill adatta la profondità e il punto di partenza in base a ciò che chiede l'utente.

| Richiesta utente | Comportamento |
|-----------------|---------------|
| "mostrami tutto il backlog" | Parte dalle Epic, albero completo |
| "mostrami le Epic" | Solo livello 1, senza espandere |
| "espandi la Epic #1001" | Parte da #1001, mostra tutta la sua gerarchia |
| "cosa c'è sotto la Feature #1010" | Mostra Requirement/Bug e Task figli |
| "mostrami i Requirement della Feature Login" | Solo livello 3 di quella Feature |
| "mostrami il Requirement #1020" | Dettaglio di quel nodo + suoi Task |
| "tutte le Feature attive" | Feature filtrate per stato Active |
| "i Bug aperti" | Bug con stati Proposed/Blocked/Active/Resolved |
| "cosa ha in sprint la Feature #1010" | Aggiunge filtro iterazione corrente |

Se la richiesta è ambigua (es. "mostrami Login"), cerca per titolo su tutti i livelli
e chiedi all'utente quale item intende prima di espandere.

---

## Flusso principale

### FASE 1 — Identifica punto di partenza e profondità

Dalla richiesta determina:
- **Punto di partenza**: tutto il backlog, una Epic specifica, una Feature, un ID preciso
- **Profondità**: quanti livelli mostrare (1 = solo quel livello, "tutto" = fino ai Task)
- **Filtri**: stato, sprint, assegnato a, tipo work item

---

### FASE 2 — Recupera i dati via WIQL

#### Strategia di query per livello

**Tutto il backlog — solo Epic (punto di partenza)**
```
mcp_ado_wit_query_by_wiql(
  project: "PROGETTO",
  wiql: "
    SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType]
    FROM WorkItems
    WHERE [System.TeamProject] = 'PROGETTO'
      AND [System.WorkItemType] = 'Epic'
      AND [System.State] <> 'Closed'
    ORDER BY [System.Id] ASC
  "
)
```

**Figli di un item specifico (Feature sotto Epic, Requirement/Bug sotto Feature, Task sotto Req/Bug)**
```
mcp_ado_wit_query_by_wiql(
  project: "PROGETTO",
  wiql: "
    SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType]
    FROM WorkItemLinks
    WHERE [Source].[System.Id] = PARENT_ID
      AND [System.Links.LinkType] = 'System.LinkTypes.Hierarchy-Forward'
      AND [Target].[System.State] <> 'Closed'
    MODE (Recursive)
  "
)
```

**Alternativa: recupera item con parent esplicito**
```
mcp_ado_wit_query_by_wiql(
  wiql: "
    SELECT [System.Id], [System.Title], [System.State],
           [System.WorkItemType], [System.Parent]
    FROM WorkItems
    WHERE [System.Parent] = PARENT_ID
    ORDER BY [System.WorkItemType] ASC, [System.Id] ASC
  "
)
```

**Ricerca per titolo (quando la richiesta è ambigua)**
```
mcp_ado_wit_query_by_wiql(
  wiql: "
    SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType]
    FROM WorkItems
    WHERE [System.Title] CONTAINS 'PAROLA_CHIAVE'
      AND [System.WorkItemType] IN ('Epic', 'Feature', 'Requirement', 'Bug', 'Task')
    ORDER BY [System.WorkItemType] ASC, [System.Id] ASC
  "
)
```

#### Strategia di caricamento progressivo

- Se l'utente chiede "tutto", carica prima le Epic, poi per ognuna carica i figli in sequenza
- Se ci sono più di **5 Epic**, mostra prima solo le Epic e chiedi se espandere tutte o una specifica
- Se una Feature ha più di **20 item figli**, avvisa e chiedi se procedere

---

### FASE 3 — Costruisci e mostra l'albero

Assembla i dati in struttura ad albero rispettando la gerarchia CCMI.
Mostra sempre: emoji stato + tipo nodo + stato testuale + ID + titolo.

**Icone per tipo:**
- 📦 Epic
- 📋 Feature
- 📄 Requirement
- 🐛 Bug
- ✅ Task

**Dopo la visualizzazione**, mostra sempre le azioni disponibili contestualmente:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cosa vuoi fare?
  • "espandi #ID"         → mostra i figli di un nodo
  • "dettagli #ID"        → descrizione, commenti, assegnazione
  • "filtra per stato X"  → es. "solo Active", "solo Blocked"
  • "aggiungi task a #ID" → crea un nuovo item figlio ⬇
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Integrazione con azure-devops-daily-tasks ↔

Quando l'utente vuole **creare** un nuovo item all'interno della gerarchia visualizzata,
questa skill delega a `azure-devops-daily-tasks` passando il contesto necessario.

### Trigger di creazione dalla lettura

Frasi che attivano la delega:
- "aggiungi un task a questo requirement"
- "crea un bug sotto la Feature #1010"
- "voglio aggiungere un requirement alla Feature Login"
- "nuovo task qui"

### Passaggio di contesto alla skill di creazione

Quando si delega, passa automaticamente alla skill `azure-devops-daily-tasks`:

```
Contesto pre-compilato:
- parentId:        ID del nodo selezionato (es. #1020)
- parentType:      tipo del parent (es. "Requirement")
- parentTitle:     titolo del parent (es. "Implementare login con SSO")
- workItemType:    tipo del figlio da creare (es. "Task")
- iterationPath:   sprint corrente se il parent è in sprint, altrimenti backlog
- areaPath:        ereditato dal parent
```

Mostra all'utente cosa sta per passare:
```
📎 Creo un nuovo Task sotto:
   📄 [Requirement] #1020 · "Implementare login con SSO"

Procedo con la creazione? La skill di creazione ti guiderà nel dettaglio.
```

Solo dopo conferma ("sì", "procedi"), attiva il flusso di `azure-devops-daily-tasks`.

### Ritorno dopo la creazione

Dopo che `azure-devops-daily-tasks` ha creato il nuovo item, questa skill:
1. Ricarica automaticamente il ramo della gerarchia interessato
2. Mostra l'albero aggiornato con il nuovo item evidenziato con `🆕`

```
📄 [Active]  #1020 · Implementare login con SSO
    └── ✅ Task
        ├── 🟢 [Active]   #1030 · Creare endpoint /auth/sso
        ├── 🔵 [Proposed] #1031 · Scrivere test unitari SSO
        └── 🆕 [Proposed] #1045 · Aggiungere rate limiting   ← appena creato
```

---

## Filtri supportati

L'utente può combinare filtri nella richiesta in linguaggio naturale:

| Filtro | Esempio richiesta | Campo WIQL |
|--------|------------------|------------|
| Stato | "solo Active", "escludi Closed" | `[System.State]` |
| Tipo | "solo Bug", "solo Feature" | `[System.WorkItemType]` |
| Sprint | "nello sprint 5", "sprint corrente" | `[System.IterationPath]` |
| Assegnato | "assegnati a Marco", "i miei" | `[System.AssignedTo]` |
| Parola chiave | "che contengono 'pagamenti'" | `[System.Title] CONTAINS` |

I filtri si combinano: "Feature attive assegnate a Marco nello sprint corrente" è valido.

---

## Gestione casi particolari

| Caso | Comportamento |
|------|---------------|
| ID non trovato | "Non ho trovato il work item #ID. Vuoi cercare per titolo?" |
| Nodo senza figli | Mostra il nodo con `└── (nessun item figlio)` |
| Item orfano (senza parent Epic) | Raggruppa in fondo sotto `⚠️ Item senza gerarchia` |
| Backlog molto grande (>20 Epic) | Mostra le prime 10, chiedi se paginare o filtrare |
| Ricerca ambigua (più risultati) | Elenca i match trovati e chiedi quale espandere |

---

## Note

- Skill di **sola lettura**: tutti i tool usati sono di lettura (`mcp_ado_wit_query_by_wiql`, `mcp_ado_wit_get_work_item`, `mcp_ado_wit_get_work_items_batch_by_ids`)
- La **creazione** è sempre delegata a `azure-devops-daily-tasks` — mai gestita internamente
- Il campo `System.Parent` è disponibile nelle query WIQL per risalire/scendere la gerarchia
- Consulta `references/mcp-tools.md` per i parametri esatti di ogni tool MCP
