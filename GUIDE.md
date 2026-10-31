# App Inventory Pipeline

Le notebook `src/notebooks/Data_LeanIX.ipynb` récupère l’inventaire LeanIX, construit des tables dimensionnelles et exporte les résultats. Les sorties du notebook sont vides.

## Configuration

Renseignez votre workspace dans `databricks.yaml`, le sous-domaine LeanIX, le catalogue cible, le volume et le bucket S3. Le token est référencé par un scope et une clé Databricks, jamais par une valeur en clair.

Les ressources sont définies dans `resources/`. Configurez votre destination de notifications si vous utilisez l’alerte d’échec. La configuration de planification doit être vérifiée avant toute activation.

```bash
databricks bundle validate -t dev
# après configuration de votre propre environnement
databricks bundle deploy -t dev
```

Databricks et LeanIX sont nécessaires à l’exécution. Ce dépôt ne contient aucun inventaire réel, ni pipeline GitHub Actions de production.
