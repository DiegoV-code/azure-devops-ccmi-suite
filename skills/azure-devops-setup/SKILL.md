---
name: azure-devops-setup
description: >
  Configura il server MCP Azure DevOps per la sessione Cowork corrente.
  Usa questa skill PRIMA di usare qualsiasi altra skill Azure DevOps su Claude Cowork Web,
  oppure quando le skill ADO non rispondono o mostrano errori di connessione.
  Trigger frasi: "configura azure devops", "collega devops", "setup devops",
  "inizializza il plugin", "il plugin non funziona", "nessun tool ADO",
  "come mi collego ad azure devops", "devo configurare il server mcp".
  La skill chiede org e PAT, scrive il .mcp.json nella VM e verifica la connessione.
  NON scrive nulla senza conferma esplicita dell'utente.
---

# Azure DevOps — Setup per-sessione (Cowork)

Skill di onboarding per configurare il server MCP Azure DevOps su **Claude Cowork Web**.

> ⚠️ **Nota sul contesto**: su Cowork il plugin gira in una VM cloud effimera.
> La configurazione che imposti con questa skill è valida **solo per la sessione corrente**
> e va ripetuta ad ogni nuova sessione Cowork. Su Claude Code Desktop la config è invece
> persistente — modifica manualmente il `.mcp.json` del plugin una volta sola.

Consulta `references/mcp-tools.md` per il tool di verifica finale.

---

## ⛔ GUARDRAIL — Regola assoluta di conferma

**Non scrivere MAI il `.mcp.json` senza aver prima:**
1. Mostrato all'utente un'anteprima del file con il PAT **mascherato** (es. `***...***`)
2. Ricevuto una conferma esplicita: "sì", "ok", "confermo", "scrivi", "vai"

**Risposte che NON sono conferma valida:** silenzio, domande, richieste di modifica, "forse".

Il messaggio di anteprima deve sempre terminare con:
```
⚠️ Il .mcp.json della VM non è ancora stato modificato.
Scrivi "sì" o "confermo" per procedere.
```

---

## Flusso principale

### FASE 1 — Introduzione

Spiega brevemente il contesto:

```
🔧 Setup Azure DevOps MCP — sessione Cowork

Questa skill configura il server MCP Azure DevOps per questa sessione.
Avrò bisogno di:
  1. Il nome della tua organizzazione Azure DevOps
  2. Il tuo Personal Access Token (PAT)

⚠️  La configurazione è valida solo per questa sessione.
    Alla prossima sessione Cowork dovrai rilanciare questa skill.

Per generare un PAT: Azure DevOps → User Settings → Personal Access Tokens → New Token
Scope consigliati: Work Items (read/write) + tutti gli scope che ti servono.

Iniziamo: qual è il nome della tua organizzazione Azure DevOps?
(es. "beantech", "contoso", "mia-azienda")
```

---

### FASE 2 — Raccolta organizzazione

Chiedi il nome dell'organizzazione. Validazione: non deve contenere spazi, slash o `http`.

Se l'utente inserisce un URL completo (es. `https://dev.azure.com/beantech`), estrai
automaticamente il nome dell'organizzazione (`beantech`) e conferma:
```
✅ Organizzazione rilevata: beantech — corretto?
```

---

### FASE 3 — Raccolta PAT

```
Perfetto. Ora incolla il tuo Personal Access Token.

Il token verrà codificato in base64 automaticamente.
Non preoccuparti del formato: incolla il token grezzo così com'è.
```

Accetta il token grezzo (non codificato). La skill calcola internamente:
```
TOKEN_BASE64 = base64(":" + PAT_GREZZO)
```

Per calcolare il base64, usa il Bash tool con il comando:
```bash
echo -n ":TOKEN_GREZZO" | base64
```
oppure via Node.js se disponibile:
```bash
node -e "console.log(Buffer.from(':TOKEN_GREZZO').toString('base64'))"
```

Se l'utente inserisce già una stringa base64 (avvisa che sembra già codificata), usala
direttamente senza riencodarla — chiedi conferma prima.

---

### FASE 4 — Localizzazione del .mcp.json nella VM

Cerca il `.mcp.json` del plugin installato nella VM in questo ordine:

```bash
# 1. Cartella plugin installata tramite marketplace
find ~/.claude/plugins/azure-devops-ccmi-suite -name ".mcp.json" 2>/dev/null | head -1

# 2. Cartella plugin generica Claude Code
find ~/.claude -name ".mcp.json" -path "*azure-devops*" 2>/dev/null | head -1

# 3. Working directory corrente
ls .mcp.json 2>/dev/null
```

Se nessun percorso viene trovato, usa come fallback la **working directory corrente**
(`.mcp.json` nella cartella di lavoro della sessione). Comunica il percorso scelto all'utente.

---

### FASE 5 — Anteprima e conferma (GUARDRAIL ⛔)

Mostra l'anteprima del file che verrà scritto con il PAT mascherato:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 ANTEPRIMA — .mcp.json da scrivere
   Percorso: /home/user/.claude/plugins/azure-devops-ccmi-suite/.mcp.json
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

{
  "mcpServers": {
    "azure-devops": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "beantech", "-a", "pat"],
      "env": {
        "PERSONAL_ACCESS_TOKEN": "***...***"
      }
    }
  }
}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  Il .mcp.json della VM non è ancora stato modificato.
    Scrivi "sì" o "confermo" per procedere.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### FASE 6 — Scrittura del file (solo dopo conferma esplicita)

Scrivi il file con i valori reali tramite il Bash tool:

```bash
cat > /percorso/.mcp.json << 'EOF'
{
  "mcpServers": {
    "azure-devops": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "NOME_ORG", "-a", "pat"],
      "env": {
        "PERSONAL_ACCESS_TOKEN": "TOKEN_BASE64"
      }
    }
  }
}
EOF
```

Dopo la scrittura, conferma:
```
✅ .mcp.json scritto in: /percorso/.mcp.json
```

---

### FASE 7 — Riconnessione MCP

La modifica del file non fa hot-reload automatico del server MCP. Istruisci l'utente:

```
🔄 Ora devi riconnettere il server MCP Azure DevOps.

In Claude Cowork:
  1. Apri il menu dei connector / server MCP
  2. Trova il server "azure-devops"
  3. Disabilita e riabilita il server, oppure clicca "Reconnect"

Oppure: ricarica la pagina Cowork (potrebbe bastare F5 / Cmd+R).

Dimmi quando hai riconnesso il server, così verifico la connessione.
```

---

### FASE 8 — Verifica connessione

Dopo che l'utente conferma di aver riconnesso il server, chiama:

```
mcp__azure-devops__core_list_projects(organization: "NOME_ORG")
```

#### Esito positivo:
```
✅ Connessione verificata!
   Organizzazione: beantech
   Progetti disponibili: [lista progetti]

Puoi ora usare tutte le skill Azure DevOps di questa suite.
```

#### Esito negativo — diagnosi:

| Errore | Causa probabile | Azione |
|--------|----------------|--------|
| `401 Unauthorized` | PAT errato, scaduto o scope insufficienti | Rilanciare la skill con un PAT valido |
| `404 Not Found` | Nome organizzazione errato | Rilanciare la skill con il nome corretto |
| Tool MCP non disponibile | Server non riconnesso | Chiedere all'utente di ripetere la riconnessione |
| Timeout | VM impegnata / server MCP ancora in avvio | Attendere 10-15 secondi e riprovare la verifica |

---

## Note

- Il PAT grezzo non viene mai mostrato in chiaro nei messaggi; solo la versione base64 viene
  scritta nel file, e nell'anteprima compare sempre mascherata come `***...***`.
- Il `.mcp.json` scritto nella VM è visibile solo alla sessione corrente e viene cancellato
  alla fine della sessione — non persiste nel repo GitHub né in nessun archivio permanente.
- Questa skill non modifica il `.mcp.json` nel repo GitHub (quello resta con i placeholder).
