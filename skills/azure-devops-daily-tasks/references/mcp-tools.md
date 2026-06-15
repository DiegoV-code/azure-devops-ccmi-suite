# Azure DevOps MCP — Tool per Work Items, Backlog e Sprint

> Fonte: [microsoft/azure-devops-mcp](https://github.com/microsoft/azure-devops-mcp)

---

## Work Items

| Tool | Descrizione |
|------|-------------|
| `mcp_ado_wit_get_work_item` | Recupera un singolo work item tramite ID |
| `mcp_ado_wit_get_work_items_batch_by_ids` | Recupera una lista di work item tramite IDs in batch |
| `mcp_ado_wit_create_work_item` | Crea un nuovo work item in un progetto specificato |
| `mcp_ado_wit_update_work_item` | Aggiorna un work item tramite ID con i campi specificati |
| `mcp_ado_wit_update_work_items_batch` | Aggiorna work item in batch |
| `mcp_ado_wit_add_child_work_items` | Crea work item figli (subordinati) sotto un parent tramite tipo e ID |
| `mcp_ado_wit_work_items_link` | Collega più work item tra loro in operazioni batch |
| `mcp_ado_wit_work_item_unlink` | Rimuove uno o più link da un singolo work item |
| `mcp_ado_wit_add_artifact_link` | Collega artefatti del repository (commit, build, PR) ai work item |
| `mcp_ado_wit_link_work_item_to_pull_request` | Associa un work item a una pull request esistente |
| `mcp_ado_wit_my_work_items` | Recupera la lista di work item relativi all'utente autenticato |
| `mcp_ado_wit_get_work_item_type` | Recupera un tipo specifico di work item |

---

## Commenti e Revisioni

| Tool | Descrizione |
|------|-------------|
| `mcp_ado_wit_list_work_item_comments` | Recupera la lista di commenti di un work item tramite ID |
| `mcp_ado_wit_add_work_item_comment` | Aggiunge un commento a un work item tramite ID |
| `mcp_ado_wit_update_work_item_comment` | Modifica un commento esistente su un work item |
| `mcp_ado_wit_list_work_item_revisions` | Recupera la lista delle revisioni di un work item |
| `mcp_ado_wit_get_work_item_attachment` | Scarica un allegato di un work item tramite il suo ID |

---

## Backlog

| Tool | Descrizione |
|------|-------------|
| `mcp_ado_wit_list_backlogs` | Recupera la lista dei backlog per un progetto e team specificati |
| `mcp_ado_wit_list_backlog_work_items` | Accede agli item del backlog per categoria, progetto e team |

---

## Sprint / Iterazioni

| Tool | Descrizione |
|------|-------------|
| `mcp_ado_wit_get_work_items_for_iteration` | Recupera la lista di work item per un'iterazione specificata |
| `mcp_ado_work_list_iterations` | Elenca tutte le iterazioni in un progetto Azure DevOps specificato |
| `mcp_ado_work_create_iterations` | Crea nuove iterazioni in un progetto Azure DevOps specificato |
| `mcp_ado_work_list_team_iterations` | Recupera la lista di iterazioni per un team specifico |
| `mcp_ado_work_assign_iterations` | Assegna iterazioni esistenti a un team specifico |
| `mcp_ado_work_get_iteration_capacities` | Recupera la capacità di un'iterazione per tutti i team |
| `mcp_ado_work_get_team_capacity` | Recupera la capacità di un team specifico per una data iterazione |
| `mcp_ado_work_update_team_capacity` | Aggiorna la capacità di un membro del team |
| `mcp_ado_work_get_team_settings` | Accede alla configurazione del team incluse iterazioni e area path |

---

## Query WIQL

| Tool | Descrizione |
|------|-------------|
| `mcp_ado_wit_get_query` | Recupera una query tramite ID o path |
| `mcp_ado_wit_get_query_results_by_id` | Recupera i risultati di una work item query dato l'ID |
| `mcp_ado_wit_query_by_wiql` | Esegue una query WIQL (Work Item Query Language) e restituisce i work item corrispondenti |

---

## Riferimenti

- [microsoft/azure-devops-mcp](https://github.com/microsoft/azure-devops-mcp)
- [TOOLSET.md — lista completa tool](https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md)
- [Getting Started](https://github.com/microsoft/azure-devops-mcp/blob/main/docs/GETTINGSTARTED.md)
