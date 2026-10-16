# TaskFlow

Application de gestion de tâches construite avec Vue.js, Node.js, PostgreSQL et déployée progressivement avec Docker puis Kubernetes.

## Présentation du projet

TaskFlow permet de créer, consulter, modifier et supprimer des tâches. Une tâche contient un titre, un indicateur d'état et des dates de création et de mise à jour. L'interface est une application Vue.js; une API REST Express gère les opérations et stocke les données dans PostgreSQL.

Le projet documente les étapes suivantes : construction d'images Docker, orchestration locale avec Compose, déploiement Kubernetes avec Pods puis Deployments et Services, externalisation de la configuration, probes, gestion des ressources et persistance de PostgreSQL.

## Architecture applicative

```text
Navigateur
     |
     v
Frontend Vue.js servi par Nginx
     |  /api/* via proxy Nginx
     v
API REST Node.js / Express
     |
     v
PostgreSQL
```

| Composant | Responsabilités | Interfaces principales |
| --- | --- | --- |
| Front | Interface de gestion des tâches et page d'information sur l'instance API. | `/`, `/a-propos`; appels relatifs `/api/...` |
| API | Validation des entrées, routes REST, initialisation du schéma et accès à PostgreSQL. | Port `3000`; routes listées ci-dessous |
| PostgreSQL | Stockage des tâches et de leurs horodatages. | Port `5432`; base et utilisateur `taskflow` |

Routes de l'API :

| Méthode | Route | Fonction |
| --- | --- | --- |
| `GET` | `/healthz` | Vérifie que le serveur HTTP répond, sans contrôler la base. |
| `GET` | `/readyz` | Vérifie que l'API peut joindre PostgreSQL. |
| `GET` | `/api/info` | Retourne le nom, la version, l'environnement, le nom d'hôte et l'ancienneté du processus. |
| `GET` | `/api/stress?ms=500` | Génère une charge CPU bornée par `STRESS_MAX_MS`. |
| `GET` | `/api/tasks` | Liste les tâches. |
| `POST` | `/api/tasks` | Crée une tâche avec un corps JSON tel que `{"title":"Préparer la séance"}`. |
| `GET` | `/api/tasks/:id` | Retourne une tâche. |
| `PATCH` | `/api/tasks/:id` | Modifie `title` et/ou `done`. |
| `DELETE` | `/api/tasks/:id` | Supprime une tâche. |

Les titres sont non vides et limités à 200 caractères. L'API initialise la table `tasks` au démarrage et utilise un verrou PostgreSQL pour sérialiser l'initialisation lorsque plusieurs réplicas démarrent simultanément.

## Architecture Docker

Les deux Dockerfiles utilisent Node.js 24 pour leurs étapes de construction. L'image API `node:24-slim` installe les dépendances de production et lance `npm start`. Le Dockerfile Front construit les assets Vue avec Vite, puis les sert avec `nginx:alpine`. La configuration Nginx sert l'application monopage et relaie `/api/` vers `taskflow-api:3000`.

| Service Compose | Image | Port conteneur | Port publié sur l'hôte |
| --- | --- | ---: | ---: |
| `db` | `postgres:16` | `5432` | Aucun |
| `api` | `bernardkabou11/taskflow-api:1.0.0` | `3000` | `3000` |
| `front` | `bernardkabou11/taskflow-front:1.0.0` | `80` | `8080` |

```text
Navigateur :8080
         |
         v
+---------------------+     /api/*      +---------------------+
| front               | --------------> | api                 |
| Nginx :80           |                 | Node/Express :3000  |
+---------------------+                 +----------+----------+
     taskflow-net                                  |
                                                                                                 | db:5432
                                                                                                 v
                                                                            +---------------------+
                                                                            | db                  |
                                                                            | PostgreSQL :5432    |
                                                                            +----------+----------+
                                                                                                 |
                                                                            volume db_data
```

Compose crée le réseau `taskflow-net`, un healthcheck PostgreSQL (`pg_isready`) et le volume nommé `db_data`. L'API attend que la base soit saine. Les valeurs d'identification présentes dans `compose.yaml` et `.env.example` sont des valeurs de démonstration, pas des secrets à réutiliser en production.

Les images API et Front sont publiées sur Docker Hub sous `bernardkabou11/taskflow-api` et `bernardkabou11/taskflow-front`, tag `1.0.0` pour le déploiement documenté. Le tag API `2.0.0` est utilisé dans l'exercice de rollout/rollback Kubernetes.

## Architecture Kubernetes

Tous les objets applicatifs sont dans le namespace `taskflow`. L'architecture finale utilise deux réplicas pour l'API et le Front, et une seule instance PostgreSQL.

```text
Namespace: taskflow

Client
    |
    v
Service front (LoadBalancer :8080 -> :80)
    |
    v
Deployment taskflow-front (2 Pods, Nginx)
    |  proxy /api -> taskflow-api:3000
    v
Service taskflow-api (ClusterIP :3000) ---- Service alias api (:3000)
    |
    v
Deployment taskflow-api (2 Pods)
    |  DB_HOST=db:5432
    v
Service db (ClusterIP :5432)
    |
    v
StatefulSet db (1 Pod stable : db-0)
    |                         ^
    |                         | Service headless db-headless
    |                         | serviceName du StatefulSet
    v
PVC postgres-data-db-0 (2Gi, ReadWriteOnce)
    |
    v
PV local provisionné par la StorageClass local-path
```

| Ressource | Rôle |
| --- | --- |
| Namespace `taskflow` | Isole les ressources de l'application. |
| Deployment `taskflow-front` | Maintient deux Pods Front. |
| Deployment `taskflow-api` | Maintient deux Pods API. |
| StatefulSet `db` | Maintient un Pod PostgreSQL nommé `db-0` et rattache son claim stable. |
| Service `front` | Expose Nginx en LoadBalancer sur `8080`, vers le port conteneur `80`. K3s conserve Traefik sur `80/443`. |
| Service `taskflow-api` | Expose l'API en interne sur `3000`; son nom est celui utilisé par le Nginx de l'image Front publiée. |
| Service `api` | Alias DNS ClusterIP supplémentaire vers les mêmes Pods API. |
| Service `db` | Point de connexion ClusterIP stable utilisé par l'API sur `5432`. |
| Service `db-headless` | Service sans ClusterIP associé au StatefulSet pour son identité réseau ordinale. |
| ConfigMaps `api-config`, `db-config` | Portent les paramètres non sensibles de l'API et PostgreSQL. |
| Secret `db-credentials` | Fournit le mot de passe `password` par fichier aux deux conteneurs. Aucun manifeste Secret n'est versionné. |
| PVC `postgres-data-db-0` | Demande `2Gi` en `ReadWriteOnce`; créé à partir du `volumeClaimTemplate`. |
| PV | Volume provisionné et lié au PVC. Avec `local-path`, il correspond à un emplacement sur le disque local d'un nœud k3s. |
| StorageClass `local-path` | Provisionneur dynamique choisi explicitement par le PVC. Vérifier sa présence et sa politique sur le cluster. |

Images Kubernetes : API `bernardkabou11/taskflow-api:1.0.0`, Front `bernardkabou11/taskflow-front:1.0.0` et PostgreSQL `postgres:17-alpine`. Compose utilise `postgres:16` pour sa base.

Le fichier `k8s/k3s-config.yaml` est une note de référence, pas un manifeste K3s à appliquer.

## Arborescence du projet

```text
.
├── .editorconfig
├── .env.example
├── .gitattributes
├── .gitignore
├── INSTRUCTIONS.md
├── README.md
├── compose.yaml
├── api/
│   ├── .dockerignore
│   ├── .nvmrc
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── app.js
│       ├── config.js
│       ├── db.js
│       ├── logger.js
│       ├── server.js
│       └── routes/
│           └── tasks.js
├── docs/
│   ├── notes-kubernetes.md
│   └── test-persistance.md
├── front/
│   ├── .dockerignore
│   ├── .nvmrc
│   ├── Dockerfile
│   ├── index.html
│   ├── nginx.conf
│   ├── package.json
│   ├── package-lock.json
│   ├── public/
│   │   └── favicon.svg
│   └── src/
│       ├── App.vue
│       ├── api.js
│       ├── main.js
│       ├── router.js
│       ├── style.css
│       └── views/
│           ├── AboutView.vue
│           └── TasksView.vue
└── k8s/
        ├── k3s-config.yaml
        └── app/
                ├── 00-namespace.yaml
                ├── api.yaml
                ├── configmaps.yaml
                ├── db.yaml
                └── front.yaml
```

`INSTRUCTIONS.md` contient les consignes de travail locales. `.env.example` est un exemple de valeurs, et les fichiers `.dockerignore` excluent les artefacts locaux des contextes de build.

## Déploiement avec Docker Compose

### Prérequis

- Docker Engine et le plugin Docker Compose.
- Accès à Docker Hub pour récupérer les images publiées.
- Les ports hôte `3000` et `8080` disponibles.

### Variables utilisées

Compose définit directement les variables suivantes :

| Conteneur | Variables |
| --- | --- |
| PostgreSQL | `POSTGRES_DB=taskflow`, `POSTGRES_USER=taskflow`, `POSTGRES_PASSWORD=taskflow` |
| API | `APP_ENV=production`, `HOST=0.0.0.0`, `PORT=3000`, `DB_HOST=db`, `DB_PORT=5432`, `DB_NAME=taskflow`, `DB_USER=taskflow`, `DB_PASSWORD=taskflow`, `DB_CONNECT_RETRIES=5`, `DB_CONNECT_INTERVAL_MS=2000` |

L'API accepte aussi `DB_PASSWORD_FILE` (prioritaire sur `DB_PASSWORD`), `STRESS_MAX_MS` (défaut `5000`) et `SHUTDOWN_TIMEOUT_MS` (défaut `8000`). Ces trois variables ne sont pas définies dans Compose. Le fichier `.env.example` n'est pas déclaré comme `env_file` dans Compose; ce dernier utilise les valeurs de `compose.yaml`.

### Lancer et vérifier

Depuis la racine du dépôt :

```powershell
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=50 db api front
```

Accès :

- Front : <http://localhost:8080>
- API : <http://localhost:3000>
- Liste des tâches : <http://localhost:3000/api/tasks>
- Santé API : <http://localhost:3000/healthz>
- Disponibilité API et DB : <http://localhost:3000/readyz>

Pour arrêter les conteneurs sans supprimer les données du volume nommé :

```powershell
docker compose down
```

`docker compose down -v` supprime aussi `db_data` et efface la base. Les identifiants actuels dans Compose sont destinés à l'exercice; remplace-les et utilise une gestion de secrets adaptée avant tout usage réel.

## Déploiement sur Kubernetes

### Prérequis

- Cluster k3s démarré et accessible depuis le contexte courant de `kubectl`.
- Images `bernardkabou11/taskflow-api:1.0.0` et `bernardkabou11/taskflow-front:1.0.0` accessibles aux nœuds.
- StorageClass `local-path` disponible; K3s l'installe généralement par défaut. Le PVC la référence explicitement.
- Port local `3000` libre pour les tests par port-forward.

Contrôler l'accès au cluster et le stockage avant de déployer :

```powershell
kubectl cluster-info
kubectl get nodes -o wide
kubectl get storageclass
kubectl get storageclass local-path -o yaml
```

Vérifie notamment `provisioner` et `reclaimPolicy`. Si `local-path` est absent, le PVC restera `Pending`; ne poursuis pas avant d'avoir corrigé la classe utilisée.

### Créer le namespace et le Secret

Le mot de passe est commun à l'API et PostgreSQL. Choisis une valeur locale et remplace le placeholder. N'ajoute jamais le Secret à un fichier versionné.

```powershell
kubectl apply -f k8s/app/00-namespace.yaml
kubectl create secret generic db-credentials -n taskflow --from-literal=password='<REMPLACER_PAR_UN_MOT_DE_PASSE>'
```

La valeur passée avec `--from-literal` peut être inscrite dans l'historique du shell. Les manifests montent la clé `password` comme fichier et configurent `DB_PASSWORD_FILE` pour l'API et `POSTGRES_PASSWORD_FILE` pour PostgreSQL. Un Secret Kubernetes est encodé en base64 par défaut; ce n'est pas un chiffrement au repos.

### Appliquer les manifests

La première migration depuis le Deployment PostgreSQL de la séance précédente ne copie pas les données de son disque éphémère. Supprime l'ancien Deployment avant de créer le StatefulSet. Cette commande ne supprime pas le PVC éventuel.

```powershell
kubectl delete deployment taskflow-db -n taskflow --ignore-not-found
kubectl apply -f k8s/app/configmaps.yaml
kubectl apply -f k8s/app/db.yaml
kubectl get pvc -n taskflow -w
```

Arrête le suivi avec `Ctrl+C` lorsque `postgres-data-db-0` est `Bound`. Dans un autre terminal, tu peux suivre la création du PV avec `kubectl get pv -w`. Puis attends PostgreSQL et déploie le reste :

```powershell
kubectl rollout status statefulset/db -n taskflow
kubectl apply -f k8s/app/api.yaml
kubectl apply -f k8s/app/front.yaml
kubectl rollout status deployment/taskflow-api -n taskflow
kubectl rollout status deployment/taskflow-front -n taskflow
```

### Vérifier les ressources et les services

```powershell
kubectl get pods -n taskflow -o wide
kubectl get deployments,statefulsets,services,pvc -n taskflow -o wide
kubectl get pv -o wide
kubectl get endpointslices -n taskflow
kubectl describe pod db-0 -n taskflow
kubectl logs db-0 -n taskflow
kubectl exec db-0 -n taskflow -- pg_isready -U taskflow -d taskflow
kubectl describe secret db-credentials -n taskflow
```

`kubectl describe secret` indique la clé sans afficher sa valeur. La commande suivante expose le Service Front; selon la configuration ServiceLB du cluster, l'adresse peut être l'IP d'un nœud :

```powershell
kubectl get service front -n taskflow -o wide
```

### Tester l'API

Laisse ce port-forward ouvert dans un terminal :

```powershell
kubectl port-forward service/taskflow-api -n taskflow 3000:3000
```

Dans un autre terminal PowerShell, contrôle santé, disponibilité et lecture :

```powershell
curl.exe http://127.0.0.1:3000/healthz
curl.exe http://127.0.0.1:3000/readyz
curl.exe http://127.0.0.1:3000/api/tasks
```

Exemple d'ajout d'une tâche :

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:3000/api/tasks -ContentType 'application/json' -Body '{"title":"Vérifier le déploiement"}'
```

## Tests réalisés

| Domaine | Vérifications documentées |
| --- | --- |
| Docker Compose | Démarrage des services, état/healthcheck PostgreSQL, endpoints API, opérations CRUD et conservation via le volume `db_data` après redémarrage des conteneurs. |
| Kubernetes | État Pods/Services/EndpointSlices, logs, `pg_isready`, DNS interne, endpoints `/healthz` et `/readyz`, opérations CRUD via port-forward. |
| Disponibilité | Suppression d'un Pod API ou Front et observation de son remplacement par le contrôleur. |
| Mise à jour | Rollout de l'API de `1.0.0` vers `2.0.0`, observation des Pods/images, puis rollback avec `kubectl rollout undo`. |
| Persistance | Suppression du Pod PostgreSQL, redémarrage du StatefulSet, suppression/recréation du StatefulSet avec PVC conservé, puis vérification des données. |

Les vérifications sont manuelles et décrites dans les guides du dépôt; les `package.json` ne déclarent pas de script de tests automatisés.

L'exercice de mise à jour progressive de l'API vers `2.0.0` et de rollback est décrit dans [docs/notes-kubernetes.md](docs/notes-kubernetes.md). Les quatre scénarios de persistance PostgreSQL, y compris suppression du Pod, redémarrage, recréation du StatefulSet et suppression définitive du PVC, sont détaillés dans [docs/test-persistance.md](docs/test-persistance.md).

Les commandes de vérification couvrent les endpoints de santé, le CRUD via l'API, l'état des Deployments/StatefulSet et PVC/PV, les logs PostgreSQL, ainsi que le maintien d'une tâche après recréation du Pod. Les manifests n'incluent pas de suite de tests automatisés; ces validations sont manuelles.

## Gestion de la configuration

Les ConfigMaps contiennent uniquement les valeurs non sensibles :

| ConfigMap | Clés principales |
| --- | --- |
| `api-config` | `APP_ENV`, `HOST`, `PORT`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, paramètres de tentatives DB |
| `db-config` | `POSTGRES_DB`, `POSTGRES_USER` |

Le Secret `db-credentials` a une clé `password`, montée dans les deux conteneurs sous `/run/secrets/db-credentials/password`. L'API lit `DB_PASSWORD_FILE`; l'image PostgreSQL 17 lit `POSTGRES_PASSWORD_FILE`. Cette méthode évite de mettre le secret comme valeur d'environnement dans les manifests. Les Pods doivent être redémarrés après modification d'une ConfigMap utilisée en variables d'environnement. Le mot de passe PostgreSQL n'est consommé qu'à l'initialisation d'un répertoire de données vide.

## Sondes Kubernetes

| Composant | Startup | Readiness | Liveness |
| --- | --- | --- | --- |
| API | HTTP `/healthz` | HTTP `/readyz` | HTTP `/healthz` |
| Front | HTTP `/` | HTTP `/` | HTTP `/` |
| PostgreSQL | `pg_isready` local | `pg_isready` local | `pg_isready` local |

La startup probe protège les démarrages lents. La readiness détermine si un Pod reçoit du trafic; pour l'API, `/readyz` inclut une vérification de PostgreSQL. La liveness détecte un processus HTTP bloqué sans dépendance à la base, afin qu'une panne DB ne provoque pas de redémarrages en boucle de l'API.

## Gestion des ressources

| Composant | Réplicas | Request CPU / mémoire par Pod | Limit CPU / mémoire par Pod | QoS |
| --- | ---: | --- | --- | --- |
| API | 2 | `100m` / `128Mi` | `400m` / `384Mi` | Burstable |
| Front | 2 | `50m` / `64Mi` | `100m` / `128Mi` | Burstable |
| PostgreSQL | 1 | `100m` / `256Mi` | `300m` / `384Mi` | Burstable |

Les requests totales sont `400m` CPU et `640Mi` mémoire; les limits totales sont `1300m` et `1408Mi`, sur une VM cible de 2 vCPU et 4 Go de RAM. La QoS est `Burstable`, car requests et limits sont définies mais différentes.

## Persistance PostgreSQL

Le StatefulSet `db` conserve une identité ordinale stable (`db-0`) et crée un PVC par ordinal via `volumeClaimTemplates`. Le Service `db-headless` est référencé par `serviceName`; le Service `db` ClusterIP reste l'adresse stable consommée par l'API. `local-path` provisionne dynamiquement un PV sur le stockage local du nœud k3s.

- **Suppression du Pod ou redémarrage :** le StatefulSet recrée `db-0` et rattache `postgres-data-db-0`; les données restent présentes.
- **Suppression/recréation du StatefulSet :** les données restent présentes tant que le PVC n'est pas supprimé et que le StatefulSet recréé utilise le même nom/template.
- **Suppression du PVC :** avec la reclaim policy habituelle `Delete` de `local-path`, le PV et les fichiers locaux peuvent être supprimés. Vérifie la politique réelle avant cette opération.
- **Suppression du namespace `taskflow` :** les PVC namespacés sont supprimés; cela peut déclencher la suppression du PV et des données.

Cette persistance protège contre le cycle de vie des Pods, pas contre la perte de la VM ou du disque. Il n'y a ni sauvegarde ni réplication PostgreSQL. Le volume est local et PostgreSQL reste une seule instance. La suppression de l'ancien Deployment lors de la migration initiale ne transfère pas ses données éphémères.

## Limites actuelles

- Stockage local à la VM/nœud k3s, sans sauvegarde automatique.
- PostgreSQL à un seul réplica, sans réplication ni haute disponibilité de la base.
- Dépendance à un cluster local k3s et à sa StorageClass `local-path`.
- Service Front de type LoadBalancer, sans Ingress ni TLS configuré pour l'application.
- Secrets Kubernetes non chiffrés au repos par une configuration spécifique du projet; la commande de création du Secret peut aussi exposer la valeur dans l'historique shell.

## Perspectives d'amélioration

- Mettre en place et tester des sauvegardes PostgreSQL.
- Étudier un opérateur PostgreSQL si les besoins de gestion et disponibilité augmentent.
- Ajouter un Ingress et TLS pour une exposition HTTP(S) adaptée.
- Ajouter du monitoring et des alertes.
- Étudier l'autoscaling pour les composants compatibles avec plusieurs réplicas.
- Automatiser build, tests et publication avec une chaîne CI/CD.

Ces pistes sont des évolutions possibles et ne sont pas déployées dans les manifests actuels.

## Auteur

À compléter.