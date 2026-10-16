# Tester la persistance PostgreSQL de TaskFlow

Ces tests s'effectuent dans le namespace `taskflow`. Ils supposent que le Secret `db-credentials` existe et que la StorageClass `local-path` est installée.

## Préparer la migration

Vérifier d'abord la classe de stockage, le provisioner et sa politique de rétention :

```powershell
kubectl get storageclass
kubectl get storageclass local-path -o yaml
```

Dans une installation k3s standard, `local-path` est la classe par défaut. Le manifeste la nomme explicitement afin que le PVC ne soit pas créé par une autre classe par défaut. Si `local-path` n'existe pas sur ce cluster, arrête-toi et corrige le manifeste ou installe/configure la StorageClass avant de continuer.

Le manifeste précédent créait PostgreSQL avec un Deployment sans volume. Supprimer ce Deployment puis appliquer le nouveau manifeste crée un stockage vide; les tâches présentes uniquement dans l'ancien Pod ne sont pas migrées.

```powershell
kubectl delete deployment taskflow-db -n taskflow --ignore-not-found
kubectl apply -f k8s/app/db.yaml
kubectl get statefulset db -n taskflow
kubectl get pvc -n taskflow -w
kubectl get pv -w
```

Attends que `postgres-data-db-0` soit `Bound` et qu'un PV correspondant apparaisse, puis arrête chaque suivi avec `Ctrl+C`. Le StatefulSet crée le PVC depuis `volumeClaimTemplates`; la StorageClass dynamique crée et lie le PV local.

```powershell
kubectl get pv -o wide
kubectl get pvc postgres-data-db-0 -n taskflow -o wide
kubectl get pod db-0 -n taskflow -o wide
kubectl rollout status statefulset/db -n taskflow
kubectl logs db-0 -n taskflow
```

Vérifier le `PGDATA` configuré par l'image et la présence du fichier de version dans le volume monté :

```powershell
kubectl exec db-0 -n taskflow -- printenv PGDATA
kubectl exec db-0 -n taskflow -- sh -c 'test "$PGDATA" = /var/lib/postgresql/data && test -s "$PGDATA/PG_VERSION" && df -h "$PGDATA"'
```

L'image `postgres:17-alpine` définit `PGDATA=/var/lib/postgresql/data` et déclare ce même chemin comme volume. Le PVC est monté directement à cet emplacement afin que `initdb` travaille dans le répertoire persistant initialement vide, sans répertoire parent intermédiaire.

Pour créer et lire des tâches, laisse ce transfert API actif dans un terminal :

```powershell
kubectl port-forward service/taskflow-api -n taskflow 3000:3000
```

Utilise les commandes `Invoke-RestMethod` indiquées ci-dessous dans un autre terminal PowerShell.

## Scénario 1 : suppression du Pod PostgreSQL

Créer une tâche et confirmer qu'elle apparaît :

```powershell
$task = Invoke-RestMethod -Method Post -Uri http://127.0.0.1:3000/api/tasks -ContentType 'application/json' -Body '{"title":"persistence-check"}'
$task
Invoke-RestMethod -Uri http://127.0.0.1:3000/api/tasks | ConvertTo-Json -Depth 5
```

Supprimer le Pod ordinal puis attendre que le StatefulSet le recrée :

```powershell
kubectl delete pod db-0 -n taskflow
kubectl wait --for=condition=Ready pod/db-0 -n taskflow --timeout=180s
kubectl get pvc postgres-data-db-0 -n taskflow
```

Relire les tâches par l'API :

```powershell
Invoke-RestMethod -Uri http://127.0.0.1:3000/api/tasks | ConvertTo-Json -Depth 5
```

La tâche `persistence-check` doit toujours être présente. Le Pod `db-0` a été recréé, mais son PVC existant a été rattaché.

## Scénario 2 : redémarrage du StatefulSet

```powershell
kubectl rollout restart statefulset/db -n taskflow
kubectl rollout status statefulset/db -n taskflow
kubectl get pvc postgres-data-db-0 -n taskflow
Invoke-RestMethod -Uri http://127.0.0.1:3000/api/tasks | ConvertTo-Json -Depth 5
```

La tâche doit encore être présente après le redémarrage du Pod.

## Scénario 3 : suppression puis recréation du StatefulSet

Supprimer le StatefulSet seulement. Ne supprime pas le PVC :

```powershell
kubectl delete statefulset db -n taskflow
kubectl get pvc postgres-data-db-0 -n taskflow
kubectl get pv -o wide
```

Le PVC doit rester `Bound`. Recréer le StatefulSet et ses Services depuis le manifeste :

```powershell
kubectl apply -f k8s/app/db.yaml
kubectl rollout status statefulset/db -n taskflow
kubectl get pvc postgres-data-db-0 -n taskflow
kubectl get pv -o wide
kubectl logs db-0 -n taskflow
Invoke-RestMethod -Uri http://127.0.0.1:3000/api/tasks | ConvertTo-Json -Depth 5
```

Le StatefulSet recrée `db-0` et réutilise le claim `postgres-data-db-0`; la tâche doit être conservée. Supprimer un StatefulSet seul ne supprime normalement pas les PVC créés par son `volumeClaimTemplates`.

## Scénario 4 : suppression définitive des données

Examiner la politique effective sur le PV lié au claim :

```powershell
kubectl get pvc postgres-data-db-0 -n taskflow -o jsonpath="{.spec.volumeName}"
kubectl get pv <NOM_DU_PV> -o jsonpath="{.spec.persistentVolumeReclaimPolicy}"
```

Avec la StorageClass k3s `local-path` standard, `reclaimPolicy` est généralement `Delete`. Supprimer le PVC libère le PV; avec cette politique, le provisioner supprime le PV et le répertoire de données local correspondant. Cette action détruit les données :

```powershell
kubectl delete pvc postgres-data-db-0 -n taskflow
```

Supprimer le namespace `taskflow` supprime également les PVC qu'il contient et peut donc entraîner cette suppression définitive. Ne supprime pas le PVC ni le namespace si tu veux conserver la base. La suppression du Pod ou du StatefulSet seul n'est pas l'action à utiliser pour effacer les données.

`local-path` stocke les données sur le disque local d'un nœud k3s : ce mécanisme protège contre le remplacement du Pod, pas contre la perte de la VM ou du disque. Il ne s'agit ni d'une sauvegarde ni d'une réplication.
