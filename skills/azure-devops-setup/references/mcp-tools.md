# MCP Tools — azure-devops-setup

Tool usato nella Fase 8 (verifica connessione).

---

## core_list_projects

Elenca i progetti nell'organizzazione Azure DevOps. Usato come "ping" per verificare
che il server MCP sia connesso e autenticato correttamente.

**Tool name:** `mcp__azure-devops__core_list_projects`

| Parametro | Tipo | Obbligatorio | Descrizione |
|-----------|------|--------------|-------------|
| `organization` | string | sì | Nome organizzazione Azure DevOps (es. `"beantech"`) |

**Esito atteso:** lista di oggetti progetto con `id`, `name`, `description`.
**Esito errore 401:** PAT non valido o scaduto.
**Esito errore 404:** organizzazione non trovata.
