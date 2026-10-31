# Runbook — alerte Teams sur échec de job Databricks (asset bundle)

À donner tel quel à Claude dans une nouvelle session ouverte sur le projet cible.
Contexte : org portfolio, workspace ne permettant pas les notification destinations
de type webhook — l'alerte doit passer par un SQL Alert v2 + une notification
destination de type Email pointant vers l'adresse générée par le canal Teams.

## Pré-requis à rassembler avant de commencer

- [ ] Nom et clé du job Databricks existant dans le bundle (resources/*.job.yml)
- [ ] Cron du job (`quartz_cron_expression`) et sa fréquence réelle (quotidien ?
      hebdo ? bi-mensuel ?) — l'alerte doit être calée dessus, pas en polling
      continu si le job est peu fréquent
- [ ] Canal Teams créé, adresse email récupérée (⋯ sur le canal → "Obtenir
      l'adresse e-mail")
- [ ] Host(s) des workspaces dev/preprod et prod, profils CLI configurés
      (`databricks auth profiles`)
- [ ] Confirmer que `system.lakeflow` (system tables Unity Catalog) est activé
      et lisible sur le/les workspace(s)
- [ ] Un SQL warehouse existant par workspace pour exécuter l'alerte (lister
      via `databricks warehouses list --profile <p> --output json`)

## Étapes

1. **Créer la notification destination** (une fois par workspace, admin requis) :
   ```bash
   databricks notification-destinations create --profile <profile> --json '{
     "display_name": "teams-<projet>-alerts",
     "config": {"email": {"addresses": ["<adresse-canal-teams>"]}}
   }'
   ```
   Même `display_name` dans tous les workspaces → un seul lookup marche partout.

2. **Variables dans `databricks.yml`** (adapter les noms au projet) :
   ```yaml
   alert_quartz_cron_expression:
     description: >-
       Même cadence que quartz_cron_expression, décalée d'environ 30 min pour
       laisser le job terminer. À garder synchronisé manuellement.
     default: "<cron du job avec minute +30>"

   teams_alert_destination_id:
     description: notification destination Email vers le canal Teams
     lookup:
       notification_destination: teams-<projet>-alerts

   alert_warehouse_id:
     lookup:
       warehouse: <nom du warehouse par défaut>
     # à surcharger par target si le nom diffère entre workspaces

   alert_pause_status:
     description: PAUSED partout sauf en prod
     default: PAUSED
   ```
   Dans `targets.prod.variables`, ajouter `alert_pause_status: UNPAUSED` (et
   surcharger `alert_warehouse_id`/`alert_quartz_cron_expression` par target si besoin).

3. **Ressource `resources/<projet>_failure_alert.alert.yml`** :
   ```yaml
   resources:
     alerts:
       <projet>_failure_alert:
         display_name: "<nom du job>-${bundle.target} - Job Failure Alert"
         parent_path: ${workspace.root_path}   # PAS de sous-dossier custom (voir piège #2)
         warehouse_id: ${var.alert_warehouse_id}

         query_text: >-
           WITH failed AS (
             SELECT run_id, period_start_time, period_end_time, result_state, termination_code
             FROM system.lakeflow.job_run_timeline
             WHERE job_id = ${resources.jobs.<clé_du_job>.id}
               AND result_state IN ('FAILED', 'ERROR', 'TIMED_OUT', 'CANCELED')
               AND period_end_time >= current_timestamp() - INTERVAL <N> HOURS
           )
           SELECT run_id, period_start_time, period_end_time, result_state,
                  termination_code, COUNT(*) OVER () AS n_failed_runs
           FROM failed ORDER BY period_end_time DESC

         schedule:
           quartz_cron_schedule: ${var.alert_quartz_cron_expression}
           timezone_id: ${var.timezone_id}
           pause_status: ${var.alert_pause_status}

         custom_summary: >-
           🔴 [${bundle.target}] <nom job> a échoué — {{QUERY_RESULT_VALUE}} run(s) en échec

         custom_description: >-
           <p>Le job Databricks <strong><nom job>-${bundle.target}</strong> a échoué.
           {{QUERY_RESULT_VALUE}} run(s) en échec détecté(s) :</p>
           {{QUERY_RESULT_TABLE}}
           <p><a href="{{ALERT_URL}}">Voir l'alerte</a> &nbsp;|&nbsp;
           <a href="{{QUERY_URL}}">Voir la requête</a></p>

         evaluation:
           comparison_operator: GREATER_THAN
           source:
             name: n_failed_runs
           threshold:
             value:
               double_value: 0
           empty_result_state: OK
           notification:
             notify_on_ok: false
             retrigger_seconds: 0
             subscriptions:
               - destination_id: ${var.teams_alert_destination_id}   # PAS user_email (voir piège #1)
   ```
   Choisir `<N>` heures de lookback = large marge après le décalage de 30 min,
   sans jamais recouvrir le run précédent (ex: job quotidien → 2-3h ; job
   bi-mensuel → 6h).

4. Valider : `databricks bundle validate -t <target> --profile <profile>`
   pour chaque target.

5. Déployer d'abord en dev/preprod (schedule pausé), vérifier dans l'UI
   (Alerts → l'alerte) que : condition, requête, notification destination et
   schedule (paused) sont corrects. Puis déployer en prod.

## Pièges déjà rencontrés (ne pas refaire l'erreur)

1. **`user_email` dans `subscriptions` exige un compte utilisateur du
   workspace.** L'adresse email générée par un canal Teams n'en est pas un →
   échec au déploiement (`Failed to get user id for email`). Toujours utiliser
   `destination_id` + une notification destination Email créée à l'avance.

2. **`parent_path: ${workspace.root_path}/alerts`** échoue si ce sous-dossier
   n'existe pas déjà dans le workspace (le bundle ne le crée pas
   automatiquement, contrairement à `workspace.root_path` lui-même). Utiliser
   `parent_path: ${workspace.root_path}` directement, sans sous-dossier.

3. **`resources.notification_destinations` n'existe pas** comme type de
   ressource bundle (CLI v0.294 testée) — la destination doit être créée via
   `databricks notification-destinations create` (admin requis), puis
   référencée par un variable `lookup: notification_destination: <nom>`.

4. **`custom_summary`/`custom_description` sont au même niveau que
   `evaluation`**, pas imbriqués dedans — vérifier `databricks bundle schema`
   si un champ semble refusé.

5. **Ne pas faire un schedule horaire/continu** si le job sous-jacent tourne
   rarement (hebdo/bi-mensuel) : caler l'alerte sur le même cron que le job,
   décalé de ~30 min, avec une fenêtre de lookback dimensionnée en
   conséquence (évite le polling inutile et les notifications dupliquées).

6. **Téléchargement Terraform qui échoue** (`unable to verify checksums
   signature: openpgp: key expired`) lors du premier `bundle deploy` sur un
   nouveau target : si un autre target du même projet a déjà un binaire
   Terraform mis en cache dans `.databricks/bundle/<autre_target>/bin/terraform`,
   le copier dans `.databricks/bundle/<target>/bin/terraform` (+`chmod +x`)
   plutôt que retenter le téléchargement.

7. **Warehouse par défaut absent selon le workspace** : ne pas supposer qu'un
   warehouse porte le même nom partout. Lister avant de choisir :
   `databricks warehouses list --profile <p> --output json`.

## Placeholders disponibles dans custom_summary / custom_description

`{{ALERT_NAME}}`, `{{ALERT_STATUS}}`, `{{ALERT_CONDITION}}`,
`{{ALERT_THRESHOLD}}`, `{{ALERT_COLUMN}}`, `{{ALERT_URL}}`, `{{QUERY_NAME}}`,
`{{QUERY_URL}}`, `{{QUERY_RESULT_TABLE}}`, `{{QUERY_RESULT_VALUE}}`,
`{{QUERY_RESULT_ROWS}}`, `{{QUERY_RESULT_COLS}}`. HTML autorisé : `table`,
`tr`, `td`, `th`, `a`, `div`, `span`, `p`, `strong`, `em`.

## Référence

Implémentation de référence : projet `process_enerdatics` dans le repo
`StratMnA`, fichiers `databricks.yml` et
`resources/process_3_failure_alert.alert.yml` (branche `process3_enerdatics`).
