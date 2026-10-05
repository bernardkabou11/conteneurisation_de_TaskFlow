📦 Conteneurisation de TaskFlow
Application full‑stack conteneurisée (Front + API + PostgreSQL)

🧩 Description du projet
TaskFlow est une application de gestion de tâches composée de :

Un front (Nginx + HTML/CSS/JS)

Une API Node.js (Express)

Une base de données PostgreSQL

L’objectif du TP est de :

créer les Dockerfiles du front et de l’API

conteneuriser PostgreSQL

orchestrer l’ensemble via Docker Compose

publier les images sur Docker Hub

garantir un fonctionnement sans build local (production)

🏗️ Architecture globale

+-------------------+        +-------------------+        +----------------------+
|     FRONT         | -----> |       API         | -----> |      PostgreSQL      |
|  (Nginx, port 80) |        | (Node.js, 3000)   |        |   (port 5432)        |
+-------------------+        +-------------------+        +----------------------+
           \____________________ Réseau Docker _____________________/


Les trois services communiquent via un réseau Docker dédié : taskflow-net.

🐳 Services Docker
1. 🗄️ Base de données — PostgreSQL
Image : postgres:16

Volume persistant : db_data

Healthcheck pour garantir que l’API démarre uniquement lorsque la DB est prête

Variables d’environnement pour initialiser la base

2. ⚙️ API — Node.js (Express)
Image Docker Hub : bernardkabou11/taskflow-api:1.0.0

Connexion à PostgreSQL via DB_HOST=db

Attente automatique de la DB grâce à depends_on: service_healthy

Exposition sur localhost:3000

3. 🎨 Front — Nginx
Image Docker Hub : bernardkabou11/taskflow-front:1.0.0

Exposition sur localhost:8080

Proxy /api vers l’API

Application statique servie par Nginx

📁 Structure du projet

conteneurisation_de_TaskFlow/
│
├── front/
│   ├── Dockerfile
│   └── (fichiers du front)
│
├── api/
│   ├── Dockerfile
│   ├── src/
│   └── package.json
│
├── compose.yaml
└── README.md


🚀 Lancement du projet (Production)
1️⃣ Démarrer la stack
docker compose up -d


➡️ Docker télécharge automatiquement les images depuis Docker Hub :

bernardkabou11/taskflow-api:1.0.0

bernardkabou11/taskflow-front:1.0.0

2️⃣ Vérifier les conteneurs
docker compose logs api --tail=50

3️⃣ Vérifier les logs API
docker compose logs api --tail=50

🌐 Accès à l’application
Front : http://localhost:8080

API : http://localhost:3000/api/tasks

🧪 Tests rapides
Ajouter une tâche

Cocher une tâche

Supprimer une tâche

Redémarrer la stack → vérifier la persistance

Vérifier les logs API :
docker compose logs api --tail=50

🛠️ Arrêter les conteneurs
docker compose down

🐳 Images Docker Hub
Les images utilisées dans ce projet sont publiées sur Docker Hub :

API : bernardkabou11/taskflow-api:1.0.0

Front : bernardkabou11/taskflow-front:1.0.0

Elles permettent un déploiement sans build local, conforme aux exigences du TP.

🎯 Objectifs pédagogiques atteints
Création de Dockerfiles (front + API)

Mise en place d’une base PostgreSQL conteneurisée

Configuration d’un réseau Docker dédié

Orchestration via Docker Compose

Gestion des dépendances (healthcheck)

Publication des images sur Docker Hub

Déploiement local stable via WSL

Application full‑stack fonctionnelle dans des conteneurs

Persistance des données validée

👤 Auteur
Bernard Daniel Kabou  
Master 2 – Clusteurisation de conteneurs
2026