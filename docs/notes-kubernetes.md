# TaskFlow Kubernetes : séances 5 et 6

Les sections précédentes conservent l'état et les manipulations des séances 5 et 6. La section « Séance 7 » décrit la configuration actuelle de PostgreSQL avec un StatefulSet et un stockage persistant local.

## Architecture et sélection

| Composant | Deployment | Réplicas | Image | Service |
| --- | --- | ---: | --- | --- |
| PostgreSQL | `taskflow-db` | 1 | `postgres:17-alpine` | `db:5432` (ClusterIP) |
| API | `taskflow-api` | 2 | `bernardkabou11/taskflow-api:1.0.0` | `taskflow-api:3000` (ClusterIP), alias `api:3000` |
| Front | `taskflow-front` | 2 | `bernardkabou11/taskflow-front:1.0.0` | `front:8080` (LoadBalancer vers le port 80 du conteneur) |

Tous les objets applicatifs sont dans le namespace `taskflow`. Chaque Deployment sélectionne des Pods par `app: taskflow` et son label `component` (`db`, `api` ou `front`); le Service correspondant reprend exactement ces sélecteurs. Les Services `api` et `taskflow-api` sélectionnent le même ensemble de Pods API.

Les images API et Front utilisent le chemin Docker Hub publié indiqué par Compose. Le manifeste conserve donc le compte `bernardkabou11`, même si les consignes abrègent le nom en `taskflow-api:1.0.0` et `taskflow-front:1.0.0`.

Le Service PostgreSQL s'appelle `db`, ce qui permet à l'API d'utiliser `DB_HOST=db`. L'image Front actuellement publiée contient `proxy_pass http://taskflow-api:3000` dans `nginx.conf`; le Service `taskflow-api` est conservé pour qu'elle fonctionne sans reconstruction. Le deuxième Service `api` est un alias DNS demandé pour la séance et permet également aux Pods du namespace de joindre l'API par `http://api:3000`.

Le Service Front écoute sur `8080` et cible le port `80` de Nginx. K3s utilise Traefik sur les ports hôte `80` et `443`; le port `8080` évite de demander à ServiceLB de réserver les ports utilisés par Traefik. `k8s/k3s-config.yaml` est une note de référence pour l'installation déjà présente, pas un manifeste à appliquer ni une demande de modifier les paramètres K3s.

## Migration et déploiement

Exécute ces commandes depuis la racine du dépôt dans PowerShell, avec le contexte `kubectl` du cluster k3s actif.

### 1. Nettoyer les Pods de la séance 4

La commande retire uniquement les Pods créés directement à la séance précédente. Elle ne supprime pas le namespace `taskflow`.

```powershell
kubectl delete pod taskflow-api taskflow-postgres -n taskflow --ignore-not-found
```

### 2. Appliquer les nouveaux objets

```powershell
kubectl apply -f k8s/app/00-namespace.yaml
kubectl apply -f k8s/app/db.yaml
kubectl rollout status deployment/taskflow-db -n taskflow
kubectl exec deployment/taskflow-db -n taskflow -- pg_isready -U taskflow -d taskflow
kubectl apply -f k8s/app/api.yaml
kubectl apply -f k8s/app/front.yaml
kubectl rollout status deployment/taskflow-api -n taskflow
kubectl rollout status deployment/taskflow-front -n taskflow
```

La commande `pg_isready` confirme que PostgreSQL accepte les connexions avant d'appliquer l'API. Aucun stockage persistant n'est monté : les données PostgreSQL restent éphémères.

### 3. Vérifier les Deployments, Pods, Services et EndpointSlices

```powershell
kubectl get deployments -n taskflow -o wide
kubectl get replicasets -n taskflow
kubectl get pods -n taskflow -o wide --show-labels
kubectl get services -n taskflow -o wide
kubectl get endpointslices -n taskflow -l kubernetes.io/service-name=db
kubectl get endpointslices -n taskflow -l kubernetes.io/service-name=taskflow-api
kubectl get endpointslices -n taskflow -l kubernetes.io/service-name=api
kubectl get endpointslices -n taskflow -l kubernetes.io/service-name=front
```

Une EndpointSlice doit référencer les adresses des Pods dont les labels correspondent au selector du Service. Si elle ne contient aucun endpoint, compare les labels du Pod template avec le selector et vérifie que les Pods sont démarrés.

### 4. Consulter les logs et tester PostgreSQL

```powershell
kubectl logs deployment/taskflow-db -n taskflow --all-pods=true --tail=50
kubectl logs deployment/taskflow-api -n taskflow --all-pods=true --tail=50
kubectl logs deployment/taskflow-front -n taskflow --all-pods=true --tail=50
kubectl exec deployment/taskflow-db -n taskflow -- pg_isready -U taskflow -d taskflow
```

### 5. Tester le DNS Kubernetes

Ces commandes créent un Pod de diagnostic temporaire BusyBox, puis le suppriment à sa sortie. Elles vérifient les deux noms API ainsi que PostgreSQL depuis le namespace `taskflow`.

```powershell
kubectl run dns-check-db -n taskflow --image=busybox:1.36 --restart=Never --rm -i -- nslookup db
kubectl run dns-check-api -n taskflow --image=busybox:1.36 --restart=Never --rm -i -- nslookup taskflow-api
kubectl run dns-check-api-alias -n taskflow --image=busybox:1.36 --restart=Never --rm -i -- nslookup api
```

Dans le même namespace, les noms courts `db`, `api` et `taskflow-api` sont résolus par le DNS Kubernetes en adresses ClusterIP. Les noms complets sont respectivement `db.taskflow.svc.cluster.local`, `api.taskflow.svc.cluster.local` et `taskflow-api.taskflow.svc.cluster.local`.

### 6. Tester l'API et le Front par port-forward

Dans un terminal, garde le port-forward de l'API actif :

```powershell
kubectl port-forward service/taskflow-api -n taskflow 3000:3000
```

Dans un autre terminal PowerShell :

```powershell
curl.exe http://127.0.0.1:3000/healthz
curl.exe http://127.0.0.1:3000/api/tasks
```

Pour tester le Front indépendamment du LoadBalancer, ouvre un second port-forward :

```powershell
kubectl port-forward service/front -n taskflow 8080:8080
```

Puis visite `http://127.0.0.1:8080`. Le Service LoadBalancer peut aussi être examiné avec `kubectl get service front -n taskflow -o wide`; sur k3s, teste `http://<IP-DU-NOEUD>:8080` si ServiceLB a publié cette adresse.

## DNS, Services et Deployments

Un Pod reçoit une IP qui peut changer à sa recréation. Un Service donne un point d'accès stable et sélectionne les Pods via des labels. Le contrôleur Service met à jour ses EndpointSlices lorsque les Pods sélectionnés changent; l'application cliente continue donc à utiliser le nom DNS au lieu de suivre les IP individuellement.

Le DNS interne publie un nom à partir du nom du Service et du namespace, sous la forme `<service>.<namespace>.svc.cluster.local`. Les Pods du même namespace peuvent généralement utiliser le nom court. Ainsi l'API se connecte à `db:5432`; Nginx contacte `taskflow-api:3000` (et `api:3000` fonctionne aussi grâce au Service alias).

Un Deployment déclare l'état souhaité, par exemple deux replicas API. Il crée et pilote un ReplicaSet, qui maintient le nombre voulu de Pods correspondant à son template. Si un Pod est supprimé, le ReplicaSet en crée un autre. Lors d'une mise à jour d'image, le Deployment crée un nouveau ReplicaSet et orchestre le remplacement progressif; l'historique permet ensuite le rollback.

Ces objets remplacent les limites de la séance précédente : le Pod API n'a plus une IP PostgreSQL inscrite en dur, et les Deployments recréent les Pods supprimés. Les Services stabilisent le point de connexion et distribuent les connexions aux Pods sélectionnés. Cela ne rend cependant pas la base persistante.

## Tester l'auto-réparation

Repère un nom concret de Pod puis supprime un seul Pod à la fois :

```powershell
kubectl get pods -n taskflow -l app=taskflow,component=api
kubectl delete pod <NOM_D_UN_POD_API> -n taskflow
kubectl get pods -n taskflow -l app=taskflow,component=api -w
```

Recommence pour le Front :

```powershell
kubectl get pods -n taskflow -l app=taskflow,component=front
kubectl delete pod <NOM_D_UN_POD_FRONT> -n taskflow
kubectl get pods -n taskflow -l app=taskflow,component=front -w
```

Le Deployment/ReplicaSet doit créer un Pod de remplacement et revenir au nombre de replicas déclaré. `Ctrl+C` arrête chaque observation `-w`.

## Mise à jour progressive et rollback de l'API

Passe l'image API à la version `2.0.0`, puis observe le rollout :

```powershell
kubectl set image deployment/taskflow-api api=bernardkabou11/taskflow-api:2.0.0 -n taskflow
kubectl rollout status deployment/taskflow-api -n taskflow
kubectl rollout history deployment/taskflow-api -n taskflow
kubectl get pods -n taskflow -l app=taskflow,component=api -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

Pour revenir à la révision précédente :

```powershell
kubectl rollout undo deployment/taskflow-api -n taskflow
kubectl rollout status deployment/taskflow-api -n taskflow
kubectl get pods -n taskflow -l app=taskflow,component=api -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

Le tag `2.0.0` doit être publié et accessible depuis les nœuds k3s. Le rollback restaure le template d'image de la révision précédente, généralement `1.0.0`.

## Limites restantes de la séance 5

- Le mot de passe PostgreSQL reste en clair dans le manifeste. Un Secret sera traité dans une séance ultérieure.
- Aucune readinessProbe ni livenessProbe n'est configurée; Kubernetes ne vérifie donc pas les endpoints `/readyz` ou `/healthz` pour décider du trafic ou du redémarrage.
- PostgreSQL n'a pas de volume persistant. La recréation du Pod peut faire perdre les données.
- Les variables de configuration sont écrites directement dans le manifeste, sans ConfigMap.
- Aucun Ingress n'est installé pour TaskFlow. L'accès externe au Front passe par le Service LoadBalancer k3s sur le port 8080.
- Les deux replicas API et Front améliorent la tolérance à la suppression d'un Pod, mais ne garantissent pas une disponibilité face à la panne d'un nœud ni celle de PostgreSQL, qui ne possède qu'un replica.

## Séance 6 : ConfigMaps, Secret et probes

Cette séance externalise la configuration, ajoute un Secret monté en fichier, des sondes et des budgets de ressources. Elle n'ajoute ni PersistentVolume, PersistentVolumeClaim, Ingress, HorizontalPodAutoscaler, Helm, StatefulSet, NetworkPolicy ni GitOps.

### Variables recensées et classement

L'inventaire correspond aux variables en dur dans les Deployments avant cette séance. Le Front ne déclarait aucune variable d'environnement.

| Composant | Variables | Classement |
| --- | --- | --- |
| API | `APP_ENV`, `HOST`, `PORT`, `DB_HOST`, `DB_PORT`, `DB_CONNECT_RETRIES`, `DB_CONNECT_INTERVAL_MS` | Configuration non sensible |
| API et DB | `DB_NAME` / `POSTGRES_DB`, `DB_USER` / `POSTGRES_USER` | Données partagées : mêmes base et utilisateur, noms propres à chaque image |
| API | `DB_PASSWORD` | Donnée sensible |
| PostgreSQL | `POSTGRES_PASSWORD` | Donnée sensible |
| Front | aucune | Rien à externaliser |

`api-config` et `db-config` portent les valeurs non sensibles. Chaque clé reprend exactement le nom de la variable d'environnement qu'elle alimente. Les variables reconnues par le code mais non configurées auparavant, comme `STRESS_MAX_MS` et `SHUTDOWN_TIMEOUT_MS`, ne sont pas ajoutées.

### Secret et prise en charge des mots de passe fichier

Le code de l'API lit `DB_PASSWORD_FILE` en priorité sur `DB_PASSWORD` et retire les espaces de fin du fichier. L'image officielle PostgreSQL prend en charge `POSTGRES_PASSWORD_FILE` en plus de `POSTGRES_PASSWORD`. Les deux Deployments montent donc la clé `password` du Secret `db-credentials` en fichier; ils ne reçoivent pas le mot de passe comme valeur de variable d'environnement.

Créer le Secret dans le namespace `taskflow`; aucun manifeste Secret n'est à créer ni à versionner :

```powershell
kubectl create secret generic db-credentials -n taskflow --from-literal=password='<REMPLACER_PAR_LE_MOT_DE_PASSE>'
```

Remplace le texte entre chevrons par le même mot de passe pour les deux composants. La commande peut conserver sa valeur dans l'historique du terminal; sur une machine partagée, utilise une méthode de saisie qui ne l'inscrit pas dans l'historique. Kubernetes encode les données d'un Secret en base64, ce qui ne chiffre pas le Secret au repos.

### Probes

| Composant | Startup | Readiness | Liveness |
| --- | --- | --- | --- |
| API | HTTP `/healthz` | HTTP `/readyz` | HTTP `/healthz` |
| Front | HTTP `/` sur Nginx | HTTP `/` sur Nginx | HTTP `/` sur Nginx |
| PostgreSQL | `pg_isready` local | `pg_isready` local | `pg_isready` local |

Dans les manifests, ces contrôles correspondent aux champs `startupProbe`, `readinessProbe` et `livenessProbe`.

`/healthz` confirme que le serveur HTTP répond et ne vérifie pas PostgreSQL. Startup et liveness utilisent donc cette route sans dépendance à la base. `/readyz` vérifie la connexion à PostgreSQL et sert à la readiness pour retirer l'API du trafic si elle ne peut pas traiter les requêtes. Les probes PostgreSQL exécutent `pg_isready` dans le conteneur, sans dépendance à un service externe.

L'API initialise la base avant d'ouvrir son port. La startup probe permet jusqu'à 150 secondes (30 échecs, espacés de 5 secondes), ce qui couvre les tentatives DB configurées et l'initialisation du schéma.

### Requests, limits et QoS

| Composant | Réplicas | Request CPU / mémoire | Limit CPU / mémoire | QoS |
| --- | ---: | --- | --- | --- |
| API | 2 | `100m` / `128Mi` chacun | `400m` / `384Mi` chacun | Burstable |
| Front | 2 | `50m` / `64Mi` chacun | `100m` / `128Mi` chacun | Burstable |
| PostgreSQL | 1 | `100m` / `256Mi` | `300m` / `384Mi` | Burstable |

Le total des requests est `400m` CPU et `640Mi` mémoire. Le total des limits est `1300m` CPU et `1408Mi` mémoire. Ces budgets restent sous les capacités de la VM k3s (2 vCPU, 4 Go) et laissent de la marge au système et au cluster. Ce sont des plafonds initiaux à réévaluer avec des mesures de charge.

Les conteneurs sont QoS `Burstable` : chacun a des requests et limits, mais celles-ci ne sont pas égales. Pour obtenir `Guaranteed`, requests et limits CPU et mémoire devraient être égales pour chaque conteneur.

### Déployer et vérifier

Depuis la racine du dépôt, avec le contexte k3s actif, créer le Secret avant les Pods qui le montent :

```powershell
kubectl apply -f k8s/app/00-namespace.yaml
kubectl create secret generic db-credentials -n taskflow --from-literal=password='<REMPLACER_PAR_LE_MOT_DE_PASSE>'
kubectl apply -f k8s/app/configmaps.yaml
kubectl apply -f k8s/app/db.yaml
kubectl rollout status deployment/taskflow-db -n taskflow
kubectl apply -f k8s/app/api.yaml
kubectl apply -f k8s/app/front.yaml
kubectl rollout status deployment/taskflow-api -n taskflow
kubectl rollout status deployment/taskflow-front -n taskflow
```

Vérifier ConfigMaps et Secret sans afficher sa valeur, puis les Pods :

```powershell
kubectl get configmaps api-config db-config -n taskflow
kubectl describe configmap api-config -n taskflow
kubectl describe configmap db-config -n taskflow
kubectl describe secret db-credentials -n taskflow
kubectl get pods -n taskflow -o wide
```

`kubectl describe secret` indique les clés, sans imprimer les données du Secret. Pour inspecter sondes et ressources déclarées :

```powershell
kubectl describe deployment taskflow-api -n taskflow
kubectl describe deployment taskflow-front -n taskflow
kubectl describe deployment taskflow-db -n taskflow
kubectl describe pod -l app=taskflow,component=api -n taskflow
kubectl describe pod -l app=taskflow,component=front -n taskflow
kubectl describe pod -l app=taskflow,component=db -n taskflow
kubectl get pods -n taskflow -o custom-columns=NAME:.metadata.name,CPU-REQUEST:.spec.containers[0].resources.requests.cpu,MEMORY-REQUEST:.spec.containers[0].resources.requests.memory,CPU-LIMIT:.spec.containers[0].resources.limits.cpu,MEMORY-LIMIT:.spec.containers[0].resources.limits.memory
kubectl top pods -n taskflow
kubectl get events -n taskflow --sort-by=.lastTimestamp
```

`kubectl top` nécessite Metrics Server. Les requests et limits configurées restent inspectables avec `kubectl describe` sans Metrics Server.

Tester les endpoints de l'API en gardant ce port-forward actif dans un terminal :

```powershell
kubectl port-forward service/taskflow-api -n taskflow 3000:3000
```

Dans un autre terminal :

```powershell
curl.exe http://127.0.0.1:3000/healthz
curl.exe http://127.0.0.1:3000/readyz
```

Les variables venant d'une ConfigMap sont figées dans l'environnement du conteneur au démarrage. Après modification de `api-config`, redémarrer l'API :

```powershell
kubectl rollout restart deployment/taskflow-api -n taskflow
kubectl rollout status deployment/taskflow-api -n taskflow
```

Après modification de `db-config`, redémarrer de la même manière `deployment/taskflow-db`. Modifier le Secret nécessite aussi de redémarrer les applications qui lisent le fichier au démarrage. PostgreSQL n'applique `POSTGRES_PASSWORD_FILE` qu'à l'initialisation d'un répertoire de données vide.

### Problèmes résolus et limites restantes

- **ConfigMaps** : les réglages non sensibles sont séparés des Deployments et des images.
- **Secret** : le mot de passe n'apparaît plus comme valeur dans les Deployments et est lu depuis un fichier monté.
- **Probes** : startup distingue le démarrage lent, readiness retire du trafic un composant indisponible et liveness redémarre un processus bloqué sans vérifier une dépendance externe.
- **Requests/limits** : le scheduler peut tenir compte des besoins et les plafonds bornent la consommation des conteneurs.

Restent volontairement hors périmètre :

- PostgreSQL reste sans persistance; la recréation du Pod peut faire perdre ses données.
- Le Front reste exposé via le Service `LoadBalancer` sur le port `8080`.
- Le chiffrement des Secrets Kubernetes au repos n'est pas activé/configuré ici.
- La valeur du mot de passe passée à `kubectl create secret` peut rester dans l'historique du terminal.

## Séance 7 : StatefulSet et persistance PostgreSQL

PostgreSQL passe du Deployment `taskflow-db` au StatefulSet `db`, toujours avec une seule réplique. Le StatefulSet donne au Pod une identité stable (`db-0`) et crée le claim `postgres-data-db-0` à partir de `volumeClaimTemplates`.

Deux Services ciblent les labels PostgreSQL :

- `db` reste un Service ClusterIP utilisé par l'API via `DB_HOST=db`.
- `db-headless` a `clusterIP: None` et sert de Service gouvernant au StatefulSet (`serviceName`). Il permet l'identité réseau stable du Pod, notamment `db-0.db-headless.taskflow.svc.cluster.local`.

Le PVC est une demande de stockage persistante créée automatiquement par le StatefulSet. La StorageClass `local-path` provisionne dynamiquement un PV sur le disque local d'un nœud k3s; le PV représente le stockage provisionné et le PVC s'y lie. Le manifeste choisit `ReadWriteOnce`, adapté à un disque local attaché à un seul nœud, et `2Gi`, une capacité initiale suffisante pour l'exercice.

L'image PostgreSQL 17 Alpine définit `PGDATA=/var/lib/postgresql/data` et déclare ce chemin comme volume. Le PVC est monté directement sur ce chemin, sans redéfinir `PGDATA`, pour que l'initialisation PostgreSQL écrive dans le répertoire persistant vide. `storageClassName: local-path` est explicite pour sélectionner la classe voulue même si une autre classe est définie par défaut. Vérifier le nom, le provisioner, la classe par défaut et la politique réellement installés avec :

```powershell
kubectl get storageclass
kubectl get storageclass local-path -o yaml
kubectl get pv -o wide
kubectl get pvc -n taskflow
```

Le cluster n'étant pas joignable pendant la préparation, ces valeurs doivent être confirmées sur l'instance avant application. K3s installe généralement `local-path` par défaut; si cette classe n'existe pas, le PVC restera en attente et le manifeste devra être ajusté.

### Ce qui est résolu

La suppression ou le redémarrage du Pod `db-0` ne supprime plus les données : le StatefulSet recrée le Pod et rattache le même PVC. La suppression seule du StatefulSet laisse normalement le PVC issu de `volumeClaimTemplates`; recréer le StatefulSet avec le même nom réutilise ce claim. Le Service `db` continue de fournir à l'API son adresse stable; aucune modification de l'API ou du Front n'est nécessaire.

La migration depuis l'ancien Deployment ne copie pas les données éphémères existantes : supprimer `taskflow-db` puis créer le StatefulSet initialisera un nouveau répertoire et une base vide. Les nouveaux changements de données sont persistés après cette initialisation.

### Limites restantes

- Le stockage `local-path` est local à la VM et au nœud provisionné. Il ne protège pas contre la perte de la VM ou du disque, ni contre le déplacement vers un autre nœud.
- Il n'y a pas de sauvegarde automatique; un PVC n'est pas une sauvegarde.
- PostgreSQL reste à une seule instance, sans réplication ni haute disponibilité de base.
- La StorageClass local-path standard utilise généralement `reclaimPolicy: Delete`. Supprimer le PVC peut donc supprimer le PV et son répertoire local. Supprimer le namespace supprime aussi ses PVC et peut entraîner la perte définitive des données. Vérifier la politique effective sur le PV avant toute suppression.