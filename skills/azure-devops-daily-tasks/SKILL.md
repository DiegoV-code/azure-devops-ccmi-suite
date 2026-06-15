---
name: azure-devops-daily-tasks
description: >
  Crea i work item (Task e Bug) della giornata lavorativa su Azure DevOps tramite MCP Azure DevOps.
  Usa questa skill ogni volta che l'utente vuole pianificare la giornata, creare task, aggiungere bug
  o work item su Azure DevOps — anche con descrizioni informali, in italiano o inglese, anche vaghe.
  Trigger frasi: "crea i task di oggi", "pianifica la mia giornata", "aggiungi questi task su DevOps",
  "devo lavorare su X Y Z oggi", "metti in backlog", "crea work item".
  La skill guida l'utente con domande interattive e NON crea nulla senza conferma esplicita.
---

# Azure DevOps — Daily Task Creator

Skill per creare i work item della giornata lavorativa in modo interattivo.
Usa esclusivamente i tool del MCP `microsoft/azure-devops-mcp`.
Consulta `references/mcp-tools.md` per la lista completa dei tool disponibili.

---

## ⛔ GUARDRAIL — Regola assoluta di conferma

**Non chiamare MAI `mcp_ado_wit_create_work_item` o qualsiasi altro tool MCP di scrittura senza aver prima:**
1. Mostrato all'utente il riepilogo completo di tutti i work item da creare
2. Ricevuto una conferma esplicita con parole come "sì", "ok", "confermo", "crea", "vai", "procedi"

**Risposte che NON sono conferma valida:**
- Silenzio o assenza di risposta
- Una domanda ("ma il titolo va bene?")
- Una modifica richiesta ("cambia il titolo del primo")
- Un "forse" o "quasi"

**Se l'utente modifica qualcosa dopo la proposta**, aggiorna il riepilogo e chiedi conferma di nuovo prima di creare.

Il riepilogo finale deve sempre terminare con:
```
⚠️ Nessun work item è ancora stato creato su Azure DevOps.
Scrivi "sì" o "confermo" per procedere, oppure dimmi cosa vuoi modificare.
```

---

## Flusso principale

### FASE 1 — Raccolta input iniziale

L'utente descrive liberamente cosa vuole fare oggi. Accetta qualsiasi formato.

Esempi di input validi:
- "oggi devo fixare il bug del login, fare review della PR di Marco e il meeting di planning"
- "fix login, PR review #42, planning meeting 2h, deploy staging"
- "ho un bug critico su pagamenti e devo scrivere i test per il modulo ordini"

---

### FASE 2 — Recupera contesto dal progetto

Prima di proporre qualsiasi cosa, recupera il contesto necessario:

```
// Recupera iterazione corrente del team
mcp_ado_work_list_team_iterations(project, team)
→ filtra per timeframe "current" per ottenere lo sprint attivo

// Recupera backlog disponibili (opzionale, se non già noto)
mcp_ado_wit_list_backlogs(project, team)
```

Usa questi dati per pre-compilare `iterationPath` nei work item.

---

### FASE 3 — Parsing e proposta strutturata

Analizza il testo e identifica i singoli work item. Per ognuno, proponi:

```
📋 Ho identificato questi work item:

1. 🐛 [Bug]  "Fix bug login"
   Descrizione: "Correggere il problema di autenticazione sulla pagina di login"
   Sprint: Sprint 5 (corrente) · Stima: —

2. ✅ [Task] "Review PR #42"
   Descrizione: "Revisione del codice della pull request #42 di Marco"
   Sprint: Sprint 5 (corrente) · Stima: —

3. ✅ [Task] "Meeting di planning"
   Descrizione: "Partecipazione al meeting di sprint planning"
   Sprint: Sprint 5 (corrente) · Stima: 2h

Vuoi modificare qualcosa, aggiungere dettagli, o procediamo?
```

**Regole di classificazione automatica:**
- → Bug: parole chiave `fix`, `bug`, `errore`, `crash`, `broken`, `problema`, `hotfix`, `patch`
- → Task: tutto il resto

Per verificare i tipi work item disponibili nel progetto:
```
mcp_ado_wit_get_work_item_type(project, type)
```

---

### FASE 4 — Intervista interattiva

Per ogni work item con dettagli mancanti, fai **una domanda alla volta**.

Priorità:
1. **Titolo** — se vago o > 255 caratteri
2. **Tipo** — se ambiguo tra Bug e Task
3. **Stima ore** — opzionale, l'utente può saltare con "nessuna"
4. **Descrizione** — solo per bug critici o task complessi
5. **Assegnazione** — default: utente corrente; chiedi solo se sembra per altri

Per identificare l'utente corrente:
```
mcp_ado_wit_my_work_items(project)
→ usa le info dell'utente autenticato per il campo assignedTo
```

---

### FASE 5 — Riepilogo finale con guardrail ⛔

Mostra il riepilogo completo e definitivo. **Non procedere finché non arriva conferma esplicita.**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 RIEPILOGO — Work item da creare
   Progetto: [PROGETTO] · Sprint: [SPRINT CORRENTE]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 🐛 [Bug]  "Fix bug login"
   └ Correggere il problema di autenticazione sulla pagina di login
   └ Stima: 2h · Assegnato a: te

2. ✅ [Task] "Review PR #42"
   └ Revisione del codice della pull request #42 di Marco
   └ Stima: 1h · Assegnato a: te

3. ✅ [Task] "Meeting di planning"
   └ Partecipazione al meeting di sprint planning
   └ Stima: 2h · Assegnato a: te

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  Nessun work item è ancora stato creato su Azure DevOps.
    Scrivi "sì" o "confermo" per procedere, oppure dimmi cosa modificare.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### FASE 6 — Creazione via MCP (solo dopo conferma esplicita)

Per ogni work item, chiama:

```
mcp_ado_wit_create_work_item(
  project:       "NOME_PROGETTO",
  type:          "Task" | "Bug",
  title:         "TITOLO",
  description:   "DESCRIZIONE",
  assignedTo:    "user@email.com",          // opzionale
  iterationPath: "Progetto\\Sprint N",      // sprint corrente recuperato in Fase 2
  areaPath:      "Progetto\\Area",          // opzionale
  remainingWork: N                          // ore stima, opzionale
)
```

Crea i work item **uno alla volta** in sequenza. Se uno fallisce, segnalalo subito e chiedi se continuare con i successivi prima di procedere.

#### Feedback dopo la creazione:
```
✅ Creati con successo:
• [#1234] Fix bug login       → https://dev.azure.com/.../workitems/edit/1234
• [#1235] Review PR #42       → https://dev.azure.com/.../workitems/edit/1235
• [#1236] Meeting di planning → https://dev.azure.com/.../workitems/edit/1236
```

#### Aggancio post-creazione → documentazione

Dopo il feedback, proponi (una sola volta, senza insistere) la documentazione
tramite la skill `azure-devops-document-task`:

```
📝 Vuoi aggiungere una descrizione documentata a uno di questi item?
   (obiettivo, approccio iniziale, primi passi) — oppure "no" per concludere.
```

Se l'utente accetta, delega a `azure-devops-document-task` passando gli ID
appena creati. Se rifiuta o ignora, concludi senza riproporre.

---

## Configurazione utente

La prima volta, chiedi:
- **Organization**: es. `mia-azienda`
- **Project**: es. `MyProject`
- **Team**: es. `Team Alpha` (per recuperare sprint e backlog corretti)

---

## Gestione errori

| Situazione | Azione |
|------------|--------|
| MCP non risponde | Avvisa l'utente, non riprovare automaticamente |
| Tipo work item non valido | Chiama `mcp_ado_wit_get_work_item_type` per verificare i tipi disponibili, proponi alternative |
| Titolo > 255 caratteri | Proponi versione accorciata e chiedi conferma prima di usarla |
| Creazione parzialmente fallita | Mostra quali sono andati a buon fine e quali no, chiedi come procedere |
