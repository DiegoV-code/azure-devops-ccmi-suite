# azure-devops-ccmi-suite — Plugin Claude Code

Plugin Claude Code per gestire **Azure DevOps Boards (processo CCMI)** tramite linguaggio naturale.
7 skill specializzate + 1 MCP server locale. Autore: Beantech.

---

## Cos'è

`azure-devops-ccmi-suite` è un plugin Claude Code che connette l'assistente al tuo Azure DevOps
tramite il server MCP ufficiale `@azure-devops/mcp` (avviato localmente via `npx`).
Permette di creare task, loggare ore, navigare la gerarchia, cambiare stati e generare report
**parlando in italiano o inglese**, senza aprire il browser.

---

## Struttura del repository

```
azure-devops-ccmi-suite/
├── .claude-plugin/
│   ├── plugin.json          # Identità del plugin (name, version, author)
│   └── marketplace.json     # Catalogo marketplace per la distribuzione al team
├── .mcp.json                # Configurazione MCP server (template con placeholder)
├── skills/
│   └── <nome-skill>/
│       ├── SKILL.md         # Definizione completa della skill (trigger, flusso, guardrail)
│       └── references/
│           └── mcp-tools.md # Parametri esatti dei tool MCP usati dalla skill
├── .gitignore
├── README.md                # Quickstart e istruzioni configurazione PAT
└── CLAUDE.md                # Questo file
```

**Convenzioni:**
- Ogni skill è **autonoma e indipendente** — il suo `SKILL.md` è autosufficiente.
- `SKILL.md` definisce: trigger frasi, flusso a fasi, guardrail (conferma obbligatoria), gestione errori.
- `references/mcp-tools.md` elenca i tool MCP usati con i parametri esatti — aggiornalo se cambia l'API.
- Le relazioni inter-skill (es. `daily-tasks` → `document-task`) sono descritte alla fine del `SKILL.md` padre.

---

## Le 8 skill

| Skill | Trigger tipico | Scopo |
|---|---|---|
| `azure-devops-setup` | "configura azure devops", "collega devops", "setup" | ⚙️ Configura il server MCP per la sessione Cowork corrente |
| `azure-devops-daily-tasks` | "crea i task di oggi", "pianifica la giornata" | Crea Task e Bug della giornata in modo interattivo |
| `azure-devops-document-task` | "documenta il task #X", "aggiungi descrizione" | Compila/aggiorna la descrizione di un work item via template HTML |
| `azure-devops-hierarchy-reader` | "mostra la gerarchia", "che task ho nell'epic #Y" | Navigazione read-only della gerarchia CCMI (Epic→Feature→Req→Task) |
| `azure-devops-log-hours` | "ho lavorato 3h sul #X", "logga le ore" | Registra ore lavorate (OriginalEstimate / CompletedWork / RemainingWork) |
| `azure-devops-sprint-tasks` | "mostra i task dello sprint", "task aperti del team" | Vista read-only dei task aperti dello sprint corrente, per persona |
| `azure-devops-state-transition` | "chiudi il task #X", "metti Active il #Y" | Cambia stati rispettando le transizioni valide CCMI; supporta cascata |

**Guardrail su skill di scrittura (`daily-tasks`, `document-task`, `log-hours`, `state-transition`):**
Nessun tool MCP di scrittura viene chiamato senza che l'utente abbia visto un riepilogo completo
e risposto esplicitamente con "sì", "ok", "confermo", "vai" o "procedi".

---

## Setup MCP / PAT (per-utente)

**Prerequisiti:** Node.js 20+

Il metodo di configurazione dipende dall'ambiente che usi:

### Su Claude Cowork Web

La VM di Cowork è **effimera**: la configurazione va rifatta **ad ogni sessione**.
Usa la skill dedicata — ti guida passo per passo:

```
configura azure devops
```

La skill chiede org + PAT, calcola la codifica, scrive il `.mcp.json` nella VM e verifica la connessione. Il PAT vive solo nella VM per la durata della sessione, poi viene cancellato.

### Su Claude Code (desktop / CLI)

La configurazione è **persistente** — si fa una volta sola.

1. Genera un **Personal Access Token** su Azure DevOps:
   _User settings → Personal access tokens → New Token_
   (scope consigliati: Work Items read/write, più gli scope necessari)

2. Codifica il token in base64 con prefisso `:` (due punti):
   ```powershell
   [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes(":" + "<IL_TUO_PAT>"))
   ```

3. Apri il `.mcp.json` del plugin installato e sostituisci i placeholder:
   - `YOUR_ORG_HERE` → nome dell'organizzazione Azure DevOps (es. `beantech`)
   - `YOUR_PAT_HERE` → stringa base64 ottenuta al passo 2

4. Riavvia / ricollega il plugin in Claude Code.

> **Non condividere mai il tuo PAT.** Non committare `.mcp.json` con valori reali.
> Il `.mcp.json` nel repo contiene solo placeholder.

---

## Distribuzione al team (installazione da GitHub)

Il plugin è pubblicato come **marketplace** su GitHub. Ogni membro del team può installarlo con due comandi:

```
# 1. Aggiungi il marketplace (una tantum per utente)
/plugin marketplace add DiegoV-code/azure-devops-ccmi-suite

# 2. Installa il plugin (una tantum per utente)
/plugin install azure-devops-ccmi-suite@ccmi-suite
```

**Aggiornamenti:** quando viene pubblicata una nuova versione, ogni utente esegue:
```
/plugin marketplace update
```

### Pubblicare una nuova versione (maintainer)

1. Fai le modifiche al plugin (skill, `.mcp.json`, ecc.)
2. Aggiorna `version` in `.claude-plugin/plugin.json` e in `.claude-plugin/marketplace.json`
   (usa versionamento semantico: MAJOR.MINOR.PATCH)
3. Commit e push su `main`:
   ```bash
   git add .
   git commit -m "release: v0.X.Y — <descrizione>"
   git push
   ```
4. Comunica al team di fare `/plugin marketplace update`.

---

## Convenzioni di contribuzione

### Aggiungere una nuova skill

1. Crea la cartella `skills/<nome-skill>/` (usa il prefisso `azure-devops-`)
2. Crea `SKILL.md` con frontmatter YAML (`name`, `description`, trigger frasi)
3. Crea `references/mcp-tools.md` con i tool MCP usati dalla skill
4. Se la skill usa tool MCP di scrittura: includi un **guardrail esplicito** (sezione `⛔ GUARDRAIL`)
5. Aggiorna questa tabella in `CLAUDE.md`
6. Bump versione MINOR in `plugin.json` e `marketplace.json`

### Versionamento

| Tipo di modifica | Bump |
|---|---|
| Nuova skill o MCP server | MINOR (0.X.0) |
| Fix/miglioramento skill esistente | PATCH (0.0.X) |
| Breaking change struttura/MCP | MAJOR (X.0.0) |

### Lingua

- `SKILL.md` e `CLAUDE.md`: italiano (è il contesto di deployment)
- Commenti tecnici nel codice/JSON: inglese o italiano, sii consistente nel file

---

## Evoluzione futura (roadmap)

- **MCP server remoto con OAuth**: quando Microsoft abiliterà Claude come client sul server remoto
  `https://mcp.dev.azure.com/{org}` (autenticazione Entra ID), la skill di setup non sarà più
  necessaria su Cowork — l'autenticazione avverrà tramite OAuth senza gestione manuale del PAT.
- **Skill `azure-devops-pr-review`**: navigazione e commento su Pull Request via MCP.
