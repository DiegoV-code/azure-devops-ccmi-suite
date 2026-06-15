# Q3-AI — Suite CCMI per Azure DevOps

Workspace Claude Code con 7 skill per gestire Azure DevOps Boards (progetto CCMI)
tramite linguaggio naturale in italiano o inglese.

## Guida

Installazione, configurazione e uso sono documentati nella guida sulla wiki del progetto:

**[Assistente DevOps — Guida](https://dev.azure.com/beantech/beanTech%20-%20Q3I/_wiki/wikis/beanTech---Q3Intelligence.wiki/15357/Assistente-DevOps-Guide)**

> La wiki è la fonte di verità: fai sempre riferimento a quella per i passaggi aggiornati.


## Configurazione (ogni utente usa il PROPRIO PAT)

Questo plugin avvia il server locale Azure DevOps MCP via `npx -y @azure-devops/mcp`
(richiede Node.js 20+). Dopo l'installazione vanno inseriti organizzazione e PAT
personali nel file `.mcp.json` del plugin.

Passi:
1. Genera un Personal Access Token su Azure DevOps:
   User settings -> Personal access tokens -> New Token (scope: Work Items read/write,
   piu' gli scope che ti servono). Copia il token.
2. Codifica il token in base64 nel formato richiesto dal server (prefisso due punti):
   - PowerShell:
     [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes(":" + "<IL_TUO_PAT>"))
3. Apri il `.mcp.json` del plugin e sostituisci:
   - YOUR_ORG_HERE  -> il nome della tua organizzazione Azure DevOps
   - YOUR_PAT_HERE   -> la stringa base64 ottenuta al passo 2
4. Riavvia / ricollega il plugin.

NOTA: il PAT e' una credenziale personale. Non condividere il tuo token ne' il file
.mcp.json compilato con altre persone.
