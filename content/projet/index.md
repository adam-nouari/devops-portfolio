---
title: Projet Final - Task Manager
---

# Projet Final - Task Manager

**Cours** : DevOps Data for SWE - ESIEE Paris 2025  
**Groupe** : E4DSIA  
**Repo** : [yanis-nouili/devops_base](https://github.com/yanis-nouili/devops_base)

---

## Description

Application web de gestion de tâches développée en groupe avec une architecture microservices entièrement conteneurisée. Le projet couvre l'ensemble de la chaîne DevOps : développement, conteneurisation, CI/CD et orchestration Kubernetes.

---

## Architecture
```
Browser → Nginx (port 5173) → Express API (port 3000) → PostgreSQL (port 5432)
```

3 conteneurs orchestrés par `docker-compose` avec healthchecks et dépendances entre services.

---

## Stack Technique

| Composant | Technologie |
|-----------|-------------|
| Frontend | React + Vite, servi par Nginx Alpine |
| Backend | Node.js 18 + Express + pg |
| Base de données | PostgreSQL 15 Alpine |
| Conteneurisation | Docker + docker-compose |
| CI/CD | GitHub Actions |
| Registry | Docker Hub |
| Orchestration | Kubernetes (manifestes k8s/) |

---

## Migration Base de Données : SQLite → PostgreSQL

### Problème initial
Le projet utilisait SQLite avec un fichier `database.db`. Les exigences imposaient une base de données conteneurisée.

### Solution

Remplacement de `sqlite3` par `pg` (node-postgres) avec connection pooling via variables d'environnement. Création d'un script `init.sql` pour initialiser la table et les données de démo au démarrage du conteneur.
```sql
CREATE TABLE IF NOT EXISTS tasks (
    id SERIAL PRIMARY KEY,
    matiere TEXT,
    title TEXT NOT NULL,
    description TEXT,
    priority TEXT DEFAULT 'Basse',
    status TEXT DEFAULT 'À faire'
);
```

---

## Conteneurisation

Service PostgreSQL avec healthcheck et volume persistant :
```yaml
db:
  image: postgres:15-alpine
  environment:
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: postgres
    POSTGRES_DB: tasksdb
  volumes:
    - postgres_data:/var/lib/postgresql/data
    - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres -d tasksdb"]
    interval: 10s
    retries: 5
```

---

## CI/CD Pipeline

Pipeline GitHub Actions avec PostgreSQL comme service pour les tests automatisés :

1. Checkout du code
2. Setup Node.js 18
3. Installation des dépendances
4. Tests Jest/Supertest avec PostgreSQL
5. Build et push automatique sur Docker Hub

---

## Kubernetes

Manifestes dans le dossier `k8s/` :

- `postgres-deployment.yaml` → PostgreSQL avec PersistentVolumeClaim (1Gi)
- `backend-deployment.yaml` → Backend Node.js
- `frontend-deployment.yaml` → Frontend Nginx
- `db-init-configmap.yaml` → Initialisation de la base de données

---

## Lancer le projet
```bash
git clone https://github.com/yanis-nouili/devops_base.git
cd devops_base/Project
docker-compose up --build
```

- Frontend : http://localhost:5173
- Backend API : http://localhost:3000/tasks
- Health check : http://localhost:3000/health