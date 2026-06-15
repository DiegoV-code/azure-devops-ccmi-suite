---
name: azure-devops-sprint-tasks
description: >
  Mostra i task aperti dello sprint corrente per persona, su Azure DevOps tramite MCP.
  Usa questa skill quando l'utente vuole vedere i work item aperti di uno o più membri del team,
  controllare il carico di lavoro nello sprint, sapere su cosa sta lavorando qualcuno.
  Trigger frasi: "mostrami i task aperti di X", "cosa ha in sprint Marco?", "task aperti del team",
  "chi ha cosa nello sprint", "work item aperti di Y", "carico di lavoro sprint",
  "mostra i miei task aperti", "cosa ho aperto io", "task dello sprint corrente".
  Skill di sola lettura — non modifica nulla su Azure DevOps.
---

# Azure DevOps — Sprint Task Viewer per persona

Skill di **sola lettura** per visualizzare i task aperti dello sprint corrente,
filtrabili per persona (uno specifico membro del team o tutti).
Usa esclusivamente tool MCP di lettura. Consulta `references/mcp-tools.md` per i dettagli.

---

## Stati "aperto" nel processo CCMI

Questo progetto usa il processo **CCMI**. Gli stati considerati aperti sono:

| Stato | Significato |
|-------|-------------|
| `Proposed` | Item proposto, non ancora iniziato |
| `Blocked` | In corso ma bloccato |
| `Active` | In lavorazione |
| `Resolved` | Risolto ma non ancora chiuso |

Lo stato `Closed` è l'unico escluso dalla vista "aperto" per default.

L'utente può restringere o ampliare il filtro stati specificandolo nella richiesta.
Esempi: "solo Active e Blocked", "anche i Closed", "solo quelli bloccati".

---

## Flusso principale

### FASE 1 — Interpreta la richiesta

Identifica dalla richiesta dell'utente:
- **Chi**: uno specifico membro (`"i task di Marco"`), se stesso (`"i miei task"`), tutto il team (`"task di tutti"`)
- **Filtro stati**: usa i 4 stati CCMI per default; aggiusta se l'utente specifica diversamente
- **Sprint**: usa quello corrente per default; accetta sprint diversi se specificato (`"nello sprint 4"`)

Se la persona è ambigua o non trovata, chiedi chiarimento prima di eseguire la query.

---

### FASE 2 — Recupera sprint corrente

```
mcp_ado_work_list_team_iterations(
  project: "PROGETTO",
  team:    "TEAM"
)
→ filtra per timeframe = "current" → ottieni iterationPath corrente
```

Se l'utente specifica uno sprint diverso, cerca per nome nell'elenco delle iterazioni.

---

### FASE 3 — Esegui query WIQL

#### Caso A — Persona specifica

```
mcp_ado_wit_query_by_wiql(
  project: "PROGETTO",
  wiql: "
    SELECT [System.Id], [System.Title], [System.State],
           [System.AssignedTo], [System.WorkItemType],
           [Microsoft.VSTS.Scheduling.RemainingWork]
    FROM WorkItems
    WHERE [System.TeamProject] = 'PROGETTO'
      AND [System.IterationPath] = 'ITERATION_PATH'
      AND [System.AssignedTo] = 'NOME_O_EMAIL'
      AND [System.State] IN ('Proposed', 'Blocked', 'Active', 'Resolved')
    ORDER BY [System.State] ASC, [System.Id] ASC
  "
)
```

#### Caso B — Tutto il team

```
mcp_ado_wit_query_by_wiql(
  project: "PROGETTO",
  wiql: "
    SELECT [System.Id], [System.Title], [System.State],
           [System.AssignedTo], [System.WorkItemType],
           [Microsoft.VSTS.Scheduling.RemainingWork]
    FROM WorkItems
    WHERE [System.TeamProject] = 'PROGETTO'
      AND [System.IterationPath] = 'ITERATION_PATH'
      AND [System.State] IN ('Proposed', 'Blocked', 'Active', 'Resolved')
    ORDER BY [System.AssignedTo] ASC, [System.State] ASC, [System.Id] ASC
  "
)
```

#### Caso C — Utente loggato ("i miei task")

Usa `@Me` nella query invece dell'email:
```sql
AND [System.AssignedTo] = @Me
```

---

### FASE 4 — Formatta e mostra i risultati

Raggruppa i risultati per persona, ordinati alfabeticamente. Per ogni persona mostra la lista dei task.

#### Formato output — Persona specifica:

```
👤 Marco Rossi — Sprint 5 · 4 task aperti

  🔴 [Blocked]  #1201 · Fix autenticazione SSO                    — 3h rimaste
  🟡 [Proposed] #1198 · Setup ambiente di staging                 — 4h rimaste
  🟢 [Active]   #1205 · Implementazione endpoint /orders          — 6h rimaste
  🔵 [Resolved] #1189 · Correzione bug pagina profilo             — —
```

#### Formato output — Tutto il team:

```
📋 Sprint 5 — Task aperti per persona

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
👤 Anna Bianchi — 3 task

  🟢 [Active]   #1210 · Review PR modulo pagamenti                — 2h rimaste
  🟡 [Proposed] #1215 · Documentazione API v2                     — 5h rimaste
  🔵 [Resolved] #1200 · Fix CSS mobile                            — —

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
👤 Marco Rossi — 4 task

  🔴 [Blocked]  #1201 · Fix autenticazione SSO                    — 3h rimaste
  🟡 [Proposed] #1198 · Setup ambiente di staging                 — 4h rimaste
  🟢 [Active]   #1205 · Implementazione endpoint /orders          — 6h rimaste
  🔵 [Resolved] #1189 · Correzione bug pagina profilo             — —

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
👤 Sara Verdi — 2 task

  🟢 [Active]   #1208 · Migrazione database schema v3             — 8h rimaste
  🟢 [Active]   #1211 · Test integrazione servizio notifiche      — 3h rimaste

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Totale: 9 task aperti · 3 persone coinvolte
```

#### Legenda colori stato:
- 🔴 Blocked
- 🟡 Proposed
- 🟢 Active
- 🔵 Resolved

---

### FASE 5 — Azioni rapide post-visualizzazione (opzionale)

Dopo aver mostrato i risultati, offri azioni contestuali di sola lettura:

```
Vuoi approfondire qualcosa?
• "dettagli #1201" → mostra descrizione e commenti del work item
• "task bloccati del team" → filtra solo i Blocked
• "task di Anna" → mostra solo una persona
```

Se l'utente chiede un'azione di scrittura (es. "assegna il #1201 a me", "chiudi il task"),
rimanda alla skill corretta: cambio stato → `azure-devops-state-transition`,
registrazione ore → `azure-devops-log-hours`, creazione → `azure-devops-daily-tasks`.
Quelle operazioni richiedono sempre conferma esplicita.

---

## Gestione casi particolari

| Caso | Comportamento |
|------|---------------|
| Nessun task aperto per la persona | Mostra `✅ Nessun task aperto per [Nome] nello sprint corrente.` |
| Persona non trovata nel team | Chiedi: "Non ho trovato '[nome]' nel team. Vuoi cercare per email o scegliere tra i membri?" |
| Sprint non trovato | Lista gli sprint disponibili e chiedi quale usare |
| Task senza ore rimanenti | Mostra `—` nel campo ore |
| Task non assegnati | Raggruppa in fondo sotto `👤 Non assegnati` |

---

## Note

- Skill di **sola lettura**: non chiama mai tool MCP di scrittura
- Il campo `RemainingWork` può essere null — gestirlo mostrando `—`
- Se la query restituisce > 50 item, avvisa l'utente e suggerisci di filtrare per persona
- Consulta `references/mcp-tools.md` per i parametri esatti dei tool MCP
