---
name: azure-devops-pm-report
description: >
  Genera un file Markdown di riepilogo per il PM con i progetti attivi e futuri,
  organizzato per sprint attuale e sprint successivo. Mostra la gerarchia CCMI
  (Epic → Feature → Requirement/Bug → Task), lo stato di ogni item e le risorse
  assegnate con ore rimanenti. Un file .md per sprint, salvato in una cartella
  dedicata sul filesystem locale.
  Trigger frasi: "genera il report per il PM", "crea il riepilogo sprint",
  "aggiorna il file del PM", "report progetti attivi", "riepilogo per il PM",
  "genera il markdown degli sprint", "report risorse", "situazione sprint".
---

# Azure DevOps — PM Sprint Report

Skill per generare un file `.md` di riepilogo sprint destinato al PM.
Legge i dati da Azure DevOps via MCP e scrive il file sul filesystem locale.
Sola lettura su ADO — nessuna modifica ai work item.
Consulta `references/mcp-tools.md` per i tool MCP.

---

## Struttura cartelle output

```
pm-reports/
├── sprint-attuale/
│   └── report_sprint_<N>_<DATA>.md
└── sprint-successivo/
    └── report_sprint_<N+1>_<DATA>.md
```

- La cartella base `pm-reports/` è configurabile (default: directory corrente)
- Il nome file usa il numero dello sprint e la data di generazione: `report_sprint_5_2026-06-10.md`
- Se esiste già un report per lo stesso sprint, chiedi se sovrascrivere o creare una nuova versione

---

## Flusso principale

### FASE 1 — Recupera sprint attuale e successivo

```
mcp_ado_work_list_team_iterations(
  project: "PROGETTO",
  team:    "TEAM"
)
```

Identifica:
- **Sprint attuale**: `timeframe = "current"`
- **Sprint successivo**: il prossimo in ordine cronologico dopo quello corrente

Se lo sprint successivo non esiste ancora, genera solo il report dello sprint attuale
e avvisa: `ℹ️ Non esiste ancora uno sprint successivo configurato in ADO.`

---

### FASE 2 — Recupera work item per ogni sprint

Per ogni sprint (attuale e successivo), esegui la query:

```
mcp_ado_wit_query_by_wiql(
  project: "PROGETTO",
  wiql: "
    SELECT [System.Id], [System.Title], [System.State],
           [System.WorkItemType], [System.AssignedTo],
           [System.Parent],
           [Microsoft.VSTS.Scheduling.RemainingWork]
    FROM WorkItems
    WHERE [System.TeamProject] = 'PROGETTO'
      AND [System.IterationPath] = 'ITERATION_PATH'
      AND [System.WorkItemType] IN ('Epic', 'Feature', 'Requirement', 'Bug', 'Task')
      AND [System.State] NOT IN ('Closed')
    ORDER BY [System.WorkItemType] ASC, [System.Id] ASC
  "
)
```

Poi ricostruisci la gerarchia CCMI usando il campo `System.Parent` per collegare
ogni item al suo parent, costruendo l'albero:
`Epic → Feature → Requirement/Bug → Task`

---

### FASE 3 — Calcola risorse per sprint

Per ogni sprint, aggrega i task per persona assegnata:

```
Per ogni Task/Bug con AssignedTo valorizzato:
  - raggruppa per nome persona
  - somma RemainingWork di tutti i suoi task aperti nello sprint
  - lista i titoli dei task assegnati
```

---

### FASE 4 — Genera il file Markdown

Genera un file `.md` per ogni sprint con la struttura definita sotto.

#### Template file: `report_sprint_<N>_<DATA>.md`

```markdown
# Report Sprint <N> — <NOME SPRINT>
**Progetto:** <NOME PROGETTO>
**Team:** <NOME TEAM>
**Periodo:** <DATA INIZIO> → <DATA FINE>
**Generato il:** <DATA ORA GENERAZIONE>

---

## 📊 Panoramica

| Metrica | Valore |
|---------|--------|
| Work item totali | X |
| Task attivi | X |
| Task bloccati | X |
| Task proposti | X |
| Ore rimanenti totali | Xh |
| Risorse coinvolte | X persone |

---

## 🗂️ Progetti attivi

> Work item con stato Active o Blocked nello sprint.

### 📦 <TITOLO EPIC>
**Stato:** <STATO> | **ID:** #<ID>

#### 📋 <TITOLO FEATURE>
**Stato:** <STATO> | **ID:** #<ID>

| ID | Tipo | Titolo | Stato | Assegnato a | Ore rim. |
|----|------|--------|-------|-------------|---------- |
| #1020 | 📄 Requirement | Implementare login SSO | 🟢 Active | Marco Rossi | 6h |
| #1021 | 🐛 Bug | Redirect loop login | 🔴 Blocked | Sara Verdi | 3h |

**Task per Requirement/Bug:**

**#1020 — Implementare login SSO**
| ID | Titolo | Stato | Assegnato a | Ore rim. |
|----|--------|-------|-------------|----------|
| #1030 | Creare endpoint /auth/sso | 🟢 Active | Marco Rossi | 4h |
| #1031 | Scrivere test unitari SSO | 🔵 Proposed | Marco Rossi | 2h |

---

## 🔮 Progetti futuri

> Work item con stato Proposed nello sprint (non ancora iniziati).

### 📦 <TITOLO EPIC>
...
[stessa struttura di Progetti attivi]

---

## 👥 Risorse — Chi fa cosa

| Persona | Task assegnati | Ore rimanenti totali |
|---------|---------------|----------------------|
| Marco Rossi | #1030 Creare endpoint /auth/sso, #1031 Scrivere test unitari SSO | 6h |
| Sara Verdi | #1032 Fix middleware redirect | 3h |
| Anna Bianchi | #1033 UI form profilo, #1034 Validazione form | 8h |
| *(non assegnati)* | #1035 Setup CI pipeline | 4h |

---

## ⚠️ Attenzione

> Item che richiedono attenzione del PM.

- 🔴 **Bloccati:** #1021 "Redirect loop login" — assegnato a Sara Verdi
- ⏰ **Senza stima:** #1035 "Setup CI pipeline" — ore rimanenti non impostate
- 👤 **Non assegnati:** #1035 "Setup CI pipeline"

---

*Report generato automaticamente da azure-devops-pm-report skill*
*Fonte: Azure DevOps — <ORG>/<PROGETTO>*
```

---

### FASE 5 — Scrivi i file sul filesystem

Crea la struttura di cartelle se non esiste e salva i file:

```bash
# Crea cartelle
mkdir -p pm-reports/sprint-attuale
mkdir -p pm-reports/sprint-successivo

# Salva i file
pm-reports/sprint-attuale/report_sprint_5_2026-06-10.md
pm-reports/sprint-successivo/report_sprint_6_2026-06-10.md
```

Se un file esiste già per lo stesso sprint, chiedi:
```
⚠️ Esiste già un report per Sprint 5 (report_sprint_5_2026-06-08.md).
   Cosa vuoi fare?
   1. Sovrascrivilo
   2. Crea nuova versione (report_sprint_5_2026-06-10.md)
   3. Annulla
```

---

### FASE 6 — Conferma e riepilogo

Dopo aver scritto i file, mostra un riepilogo:

```
✅ Report generati con successo:

📁 pm-reports/
├── sprint-attuale/
│   └── report_sprint_5_2026-06-10.md    (42 item · 8 risorse · 87h rimanenti)
└── sprint-successivo/
    └── report_sprint_6_2026-06-10.md    (18 item · 5 risorse · 124h rimanenti)

Il PM può trovare i file nella cartella: ./pm-reports/
```

---

## Sezione "Attenzione" — logica di popolamento

La sezione `⚠️ Attenzione` viene popolata automaticamente con:

| Condizione | Segnalazione |
|------------|-------------|
| `State = Blocked` | 🔴 Item bloccato — richiede sblocco |
| `AssignedTo` vuoto | 👤 Item non assegnato |
| `RemainingWork` = 0 o null su item Active | ⏰ Item attivo senza stima ore |
| Task con parent Closed | 🔗 Task orfano — parent già chiuso |
| Persona con > 40h rimanenti nello sprint | 📈 Possibile sovraccarico risorse |

Se non ci sono segnalazioni, la sezione mostra: `✅ Nessuna anomalia rilevata.`

---

## Legenda stati nel Markdown

| Simbolo | Stato |
|---------|-------|
| 🔵 | Proposed |
| 🟢 | Active |
| 🔴 | Blocked |
| 🟣 | Resolved |
| ⚫ | Closed |

---

## Configurazione

La prima volta chiedi e salva:
- **Cartella base output**: default `./pm-reports`
- **Organization / Project / Team**: riutilizza da configurazione condivisa se disponibile

---

## Gestione casi particolari

| Caso | Comportamento |
|------|---------------|
| Sprint senza item | Genera il file con sezioni vuote e nota: `ℹ️ Nessun work item in questo sprint.` |
| Epic/Feature senza figli nello sprint | Ometti dall'output — includi solo se ha almeno un figlio nello sprint |
| Item con parent fuori sprint | Mostra il titolo del parent tra parentesi come riferimento |
| Risorse con nomi duplicati | Usa email come discriminante |

---

## Note

- Skill di **sola lettura** su ADO — non modifica nessun work item
- Il file `.md` è scritto in **italiano**
- Le ore rimanenti mostrano `—` se il campo è null o 0 su item Proposed
- Consulta `references/mcp-tools.md` per i parametri esatti dei tool MCP
