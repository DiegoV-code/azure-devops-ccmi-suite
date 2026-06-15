---
name: azure-devops-document-task
description: >
  Compila o aggiorna la descrizione di un Task o Bug Azure DevOps usando un template
  strutturato. Si aggancia automaticamente alle skill azure-devops-daily-tasks (creazione)
  e azure-devops-log-hours (aggiornamento ore) proponendo la documentazione dopo ogni
  operazione. Può essere invocata anche standalone per documentare qualsiasi item.
  La compilazione è sempre opzionale — l'utente può saltare.
  NON scrive nulla senza conferma esplicita (guardrail).
  Trigger frasi: "documenta il task #X", "aggiungi descrizione al #X",
  "scrivi cosa ho fatto sul #X", "compila il template del #X",
  "aggiorna la descrizione", "documenta il lavoro fatto",
  "aggiungi note al task", "cosa ho fatto oggi su #X".
---

# Azure DevOps — Document Task

Skill per compilare la descrizione di Task e Bug con un template strutturato.
Si integra con `azure-devops-daily-tasks` (post-creazione) e `azure-devops-log-hours`
(post-registrazione ore). Sempre opzionale, mai bloccante.
Usa il MCP `microsoft/azure-devops-mcp`. Consulta `references/mcp-tools.md`.

---

## ⛔ GUARDRAIL — Regola assoluta di conferma

**Non chiamare MAI `mcp_ado_wit_update_work_item` senza aver prima:**
1. Mostrato l'anteprima leggibile della descrizione che verrà salvata
2. Ricevuto conferma esplicita: "sì", "ok", "confermo", "salva", "vai", "procedi"

**Risposte che NON sono conferma valida:**
silenzio, domande, richieste di modifica, "forse", "quasi".

**Se l'utente corregge qualcosa dopo l'anteprima** → rigenera l'anteprima
e chiedi conferma di nuovo da zero.

L'anteprima deve sempre terminare con:
```
⚠️ La descrizione non è ancora stata salvata su Azure DevOps.
Scrivi "sì" o "confermo" per salvare, oppure dimmi cosa modificare.
```

---

## Template di documentazione

Template unico per **Task e Bug**. Ogni sezione è opzionale tranne il titolo della entry.

```html
<!-- ENTRY [DATA] — [AUTORE] -->
<h3>📋 Cosa è stato fatto</h3>
<p>[Descrizione del lavoro svolto]</p>

<h3>🧠 Decisioni prese</h3>
<p>[Scelte tecniche o funzionali fatte durante il lavoro — lasciare vuoto se nessuna]</p>

<h3>➡️ Prossimi passi</h3>
<p>[Cosa resta da fare o cosa fare dopo — lasciare vuoto se completato]</p>

<h3>🔗 Commit / Riferimenti</h3>
<ul>
  <li>Commit: [ID commit o link — opzionale]</li>
  <li>Note: [link a PR, doc, ticket correlato — opzionale]</li>
</ul>
<!-- END ENTRY -->
```

> Il template usa HTML perché ADO renderizza la descrizione come HTML.
> Le sezioni vuote vengono omesse automaticamente nell'output finale.

---

## Modalità di aggancio

### Modalità 1 — Post-creazione (da `azure-devops-daily-tasks`)

Dopo che `daily-tasks` ha creato uno o più item con successo, questa skill
viene proposta automaticamente per ogni item appena creato:

```
✅ Task #1045 creato — "Aggiungere rate limiting"

📝 Vuoi aggiungere una descrizione documentata?
   Puoi scrivere liberamente cosa prevedi di fare, le decisioni iniziali,
   o saltare con "no" / "skip".
```

In fase di creazione il template si adatta: non ci sono ancora "cose fatte",
quindi propone le sezioni più utili in apertura:

```html
<!-- ENTRY [DATA] — [AUTORE] — Apertura task -->
<h3>📋 Obiettivo</h3>
<p>[Cosa deve fare questo task]</p>

<h3>🧠 Decisioni / Approccio iniziale</h3>
<p>[Come si prevede di affrontarlo]</p>

<h3>➡️ Prossimi passi</h3>
<p>[Primo step concreto]</p>
<!-- END ENTRY -->
```

### Modalità 2 — Post-aggiornamento ore (da `azure-devops-log-hours`)

Dopo che `log-hours` ha registrato le ore con successo, propone la documentazione
per ogni item aggiornato:

```
✅ Ore registrate su #1030 (+2h)

📝 Vuoi documentare cosa hai fatto in queste 2 ore?
   Scrivi liberamente o di' "skip" per saltare.
```

In questa modalità usa il template completo (cosa fatto, decisioni, prossimi passi, commit).

### Modalità 3 — Standalone

L'utente chiama direttamente la skill su un item qualsiasi.
La skill recupera prima la descrizione esistente e mostra il contesto:

```
📄 [Task] #1030 · "Creare endpoint /auth/sso"
   Stato: 🟢 Active

   Descrizione attuale:
   ┌─────────────────────────────────────┐
   │ [contenuto esistente o "vuota"]     │
   └─────────────────────────────────────┘

   Cosa vuoi documentare?
```

---

## Flusso principale

### FASE 1 — Recupera item e descrizione attuale

```
mcp_ado_wit_get_work_item(
  id: ID,
  fields: [
    "System.Id",
    "System.Title",
    "System.WorkItemType",
    "System.State",
    "System.Description",
    "System.AssignedTo"
  ]
)
```

Controlla se `System.Description` è vuota/null o ha già contenuto.

---

### FASE 2 — Raccolta input con intervista leggera

Chiedi le informazioni per compilare il template, **una sezione alla volta**,
in linguaggio naturale. Non mostrare il template grezzo all'utente.

#### Sequenza domande:

**Step 1 — Cosa è stato fatto** (sempre chiesto)
```
Cosa hai fatto su "[titolo]"?
Scrivi liberamente, anche in modo informale.
```

**Step 2 — Decisioni prese** (chiesto solo se step 1 non le include già)
```
Hai preso decisioni tecniche o funzionali durante il lavoro?
(es. "ho scelto di usare X invece di Y perché...") — scrivi "nessuna" per saltare
```

**Step 3 — Prossimi passi** (chiesto solo se l'item non è Closed/Resolved)
```
Cosa resta da fare? — scrivi "niente" o "completato" per saltare
```

**Step 4 — Commit** (sempre opzionale, proposto brevemente)
```
Hai un ID commit o PR da collegare? (opzionale — scrivi "no" per saltare)
```

Se l'utente scrive tutto in una volta nel primo step (es. "ho fatto X, ho deciso Y,
commit abc123"), interpreta e popola automaticamente le sezioni senza fare altre domande.

---

### FASE 3 — Compila il template

Assembla la entry HTML con:
- Data odierna nel formato `GG/MM/YYYY`
- Nome autore dall'utente loggato (recuperato dal contesto MCP)
- Solo le sezioni con contenuto — ometti le vuote

**Esempio output compilato:**
```html
<!-- ENTRY 10/06/2026 — Marco Rossi -->
<h3>📋 Cosa è stato fatto</h3>
<p>Implementato l'endpoint POST /auth/sso con validazione token JWT.
Aggiunta gestione errori per token scaduto e firma non valida.</p>

<h3>🧠 Decisioni prese</h3>
<p>Scelto di usare la libreria jose invece di jsonwebtoken per migliore
supporto agli algoritmi ES256. Documentato in README.</p>

<h3>➡️ Prossimi passi</h3>
<p>Aggiungere rate limiting sull'endpoint. Scrivere test di integrazione.</p>

<h3>🔗 Commit / Riferimenti</h3>
<ul>
  <li>Commit: <code>a3f9c21</code></li>
</ul>
<!-- END ENTRY -->
```

---

### FASE 4 — Gestione append vs replace

**Descrizione attuale vuota** → usa la entry come unica descrizione.

**Descrizione attuale con contenuto** → aggiungi in append con separatore:

```html
[contenuto esistente]

<hr/>

<!-- ENTRY 10/06/2026 — Marco Rossi -->
[nuova entry]
<!-- END ENTRY -->
```

---

### FASE 5 — Anteprima e guardrail ⛔

Mostra sempre un'anteprima leggibile (non HTML grezzo) prima di scrivere:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📝 ANTEPRIMA — Descrizione da salvare
   [Task] #1030 · "Creare endpoint /auth/sso"
   Modalità: append alla descrizione esistente
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 Cosa è stato fatto
Implementato l'endpoint POST /auth/sso con validazione token JWT.
Aggiunta gestione errori per token scaduto e firma non valida.

🧠 Decisioni prese
Scelto di usare la libreria jose invece di jsonwebtoken per migliore
supporto agli algoritmi ES256. Documentato in README.

➡️ Prossimi passi
Aggiungere rate limiting sull'endpoint. Scrivere test di integrazione.

🔗 Commit: a3f9c21

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  La descrizione non è ancora stata salvata su Azure DevOps.
    Scrivi "sì" o "confermo" per salvare, oppure dimmi cosa modificare.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### FASE 6 — Salvataggio via MCP (solo dopo conferma esplicita)

```
mcp_ado_wit_update_work_item(
  id: ID,
  fields: {
    "System.Description": "[HTML completo con append o replace]"
  }
)
```

#### Feedback dopo il salvataggio:
```
✅ Descrizione salvata su #1030 · "Creare endpoint /auth/sso"
```

---

## Gestione bulk (più item dalla stessa sessione)

Quando arriva da `log-hours` con più item aggiornati, propone la documentazione
per tutti ma gestisce uno alla volta in sequenza:

```
📝 Vuoi documentare il lavoro fatto su questi item?

  1. #1030 · "Creare endpoint /auth/sso"       — puoi documentare
  2. #1031 · "Scrivere test unitari SSO"        — puoi documentare
  3. #1032 · "Fix middleware redirect"          — puoi documentare

Iniziamo dal primo, oppure di' quali vuoi documentare (es. "solo 1 e 3")
o "skip tutti" per saltare.
```

Per ogni item segue il flusso completo (FASE 2 → 6) prima di passare al successivo.

---

## Gestione casi particolari

| Caso | Comportamento |
|------|---------------|
| Utente dice "skip" / "no" / "dopo" | Ringrazia e chiude senza fare nulla |
| Item non è Task o Bug | Avvisa: "La documentazione strutturata è disponibile solo per Task e Bug." |
| Utente scrive tutto in una volta | Interpreta e popola le sezioni senza fare altre domande |
| Descrizione esistente molto lunga | Mostra solo le ultime 2 entry precedenti nell'anteprima, non tutta la storia |
| Commit ID non valido (formato strano) | Accetta comunque — non valida il formato, è testo libero |

---

## Note

- Il template usa HTML perché ADO renderizza `System.Description` come rich text HTML
- Le entry sono delimitate da commenti `<!-- ENTRY ... -->` e `<!-- END ENTRY -->` per permettere parsing futuro
- Non modificare mai altri campi oltre a `System.Description`
- La skill non valida il contenuto — l'utente è libero di scrivere quello che vuole
- Consulta `references/mcp-tools.md` per i parametri esatti dei tool MCP
