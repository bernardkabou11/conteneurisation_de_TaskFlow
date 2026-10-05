📦 Conteneurisation de TaskFlow
Application full‑stack conteneurisée (Front + API + PostgreSQL)

🧩 Description du projet
TaskFlow est une petite application de gestion de tâches composée de :

Un front (HTML/CSS/JS)

Une API Node.js (Express)

Une base de données PostgreSQL

Ce TP consiste à conteneriser entièrement l’application, puis à orchestrer les trois services via Docker Compose.

🏗️ Architecture globale
Code
+-------------------+        +-------------------+        +----------------------+
|     FRONT         | -----> |       API         | -----> |      PostgreSQL      |
|  (Nginx, port 80) |        | (Node.js, 3000)   |        |   (port 5432)        |
+-------------------+        +-------------------+        +----------------------+
           \____________________ Réseau Docker _____________________/
Les trois conteneurs communiquent via un réseau Docker dédié : taskflow-net.

🐳 Services Docker
1. Base de données (PostgreSQL)
Image : postgres:16

Volume persistant : db_data

Healthcheck pour garantir que l’API démarre seulement quand la DB est prête

Variables d’environnement pour initialiser la base

2. API (Node.js)
Build via Dockerfile

Connexion à PostgreSQL via DB_HOST=db

Attente automatique de la DB grâce à depends_on + healthcheck

Exposition sur localhost:3000

3. Front (Nginx)
Build via Dockerfile

Exposition sur localhost:8080

Communique avec l’API via /api/tasks

📁 Structure du projet
Code
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
🚀 Lancement du projet
1. Construire et démarrer les conteneurs
Dans WSL (Ubuntu) :

Code
docker compose up -d --build
2. Vérifier les conteneurs
Code
docker compose ps
3. Vérifier les logs API
Code
docker compose logs api --tail=50
🌐 Accès à l’application
Front :
http://localhost:8080

API :
http://localhost:3000/api/tasks

🧪 Tests rapides
Ajouter une tâche

Cocher une tâche

Supprimer une tâche

Vérifier les logs API :

Code
docker compose logs api --tail=50
🛠️ Arrêter les conteneurs
Code
docker compose down
🎯 Objectifs pédagogiques atteints
Création de Dockerfiles (front + API)

Mise en place d’une base PostgreSQL conteneurisée

Configuration d’un réseau Docker

Orchestration via Docker Compose

Gestion des dépendances (healthcheck)

Déploiement local stable via WSL

Application full‑stack fonctionnelle dans des conteneurs

👤 Auteur
Bernard Daniel Kabou
Master 2 – Clusteurisation de conteneurs
2026