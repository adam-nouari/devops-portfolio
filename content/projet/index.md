---
title: Projet Final - Task Manager
---
**Github** : [yanis-nouili/devops_base](https://github.com/yanis-nouili/devops_base)

---

## Introduction et Présentation du Projet

### Contexte

Ce rapport présente le travail réalisé dans le cadre du projet final du cours DevOps à l'ESIEE Paris. L'objectif est de mettre en œuvre l'ensemble de la chaîne DevOps : du code source jusqu'à l'application déployée, en s'appuyant sur des technologies industrielles réelles — conteneurs Docker, orchestration Kubernetes, et pipeline d'automatisation via GitHub Actions.

Nous avons développé **Task Manager**, une application web de gestion de tâches permettant de créer, modifier, suivre et supprimer des tâches organisées par matière et priorité. Ce projet nous a permis d'explorer simultanément le développement applicatif, la conteneurisation, l'intégration continue et le déploiement orchestré.

### Objectifs Techniques

Le projet couvre quatre grandes dimensions. La **conteneurisation** de chaque composant via Docker avec optimisation des images. L'**intégration continue (CI)** avec automatisation des tests à chaque push. La **livraison continue (CD)** avec push automatique des images sur Docker Hub. Enfin la **gestion des données** avec une base PostgreSQL conteneurisée et persistante.

---

## Architecture Globale

### Choix de l'Architecture Microservices

L'application a été conçue selon le principe des microservices : trois composants indépendants ayant chacun une responsabilité unique, communicant entre eux au sein d'un réseau Docker.
```
Browser → Nginx/Frontend (port 5173) → Express API (port 3000) → PostgreSQL (port 5432)
```

**Le Frontend** est une interface React + Vite servie par Nginx Alpine. Il communique avec le backend via des appels HTTP et est le seul composant exposé à l'extérieur.

**Le Backend** est une API RESTful développée avec Node.js et Express. Il expose les routes CRUD pour la gestion des tâches et communique avec PostgreSQL via le driver `pg`. Il écoute sur le port 3000 et n'est accessible qu'en interne.

**La Base de données** est une instance PostgreSQL 15 Alpine déployée en conteneur avec un volume persistant. Elle écoute sur le port 5432 et est protégée derrière le réseau interne Docker.

### Tableau des Composants Techniques

| Composant | Technologie | Port | Accessibilité | Rôle |
|-----------|-------------|------|---------------|------|
| Frontend | React + Vite + Nginx Alpine | 5173 | Public | Interface utilisateur |
| Backend | Node.js 18 + Express + pg | 3000 | Interne | API REST + logique métier |
| Base de données | PostgreSQL 15 Alpine | 5432 | Interne | Persistance des tâches |
| Orchestration locale | docker-compose | — | Local | Gestion des conteneurs |
| Orchestration cloud | Kubernetes (k8s/) | — | Cluster | Déploiement production |
| Registry | Docker Hub | — | Internet | Stockage des images |
| CI/CD | GitHub Actions | — | GitHub | Automatisation |

---

## Phase 1 : Développement Applicatif

### Le Backend Node.js + Express

Le backend constitue le cœur logique de l'application. Il expose une API RESTful avec quatre routes CRUD et un endpoint de santé pour Kubernetes.

**Connexion PostgreSQL via variables d'environnement :**
```javascript
const pool = new Pool({
    host: process.env.DB_HOST || 'db',
    port: process.env.DB_PORT || 5432,
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || 'postgres',
    database: process.env.DB_NAME || 'tasksdb',
});
```

L'utilisation de variables d'environnement est un point essentiel : au lieu d'écrire les paramètres de connexion en dur, on les lit depuis l'environnement. Dans docker-compose, ces variables sont injectées dans le service `backend`. Dans Kubernetes, elles sont définies dans le manifeste YAML.

**Routes CRUD implémentées :**
```javascript
// READ : Lire toutes les tâches
app.get('/tasks', async (req, res) => {
    const result = await pool.query('SELECT * FROM tasks ORDER BY id');
    res.json(result.rows);
});

// CREATE : Créer une tâche
app.post('/tasks', async (req, res) => {
    const { matiere, title, description, priority } = req.body;
    const result = await pool.query(
        'INSERT INTO tasks (matiere, title, description, priority) VALUES ($1, $2, $3, $4) RETURNING *',
        [matiere || "Général", title, description, priority || 'Basse']
    );
    res.status(201).json(result.rows[0]);
});

// UPDATE : Modifier une tâche
app.put('/tasks/:id', async (req, res) => {
    await pool.query(
        `UPDATE tasks SET title=$1, description=$2, priority=$3, status=$4, matiere=$5 WHERE id=$6`,
        [title, description, priority, status, matiere, id]
    );
    res.json({ message: "Tâche mise à jour ✅" });
});

// DELETE : Supprimer une tâche
app.delete('/tasks/:id', async (req, res) => {
    await pool.query('DELETE FROM tasks WHERE id = $1', [req.params.id]);
    res.json({ message: "Tâche supprimée !" });
});

// HEALTH CHECK : Pour les probes Kubernetes
app.get('/health', async (req, res) => {
    await pool.query('SELECT 1');
    res.json({ status: 'ok', db: 'connected' });
});
```

Le module `app` est exporté avec `module.exports = app` pour permettre son import dans les tests Jest/Supertest sans démarrer le serveur.

### Migration SQLite → PostgreSQL

Le projet utilisait initialement SQLite. La migration vers PostgreSQL a nécessité de remplacer le driver `sqlite3` par `pg` (node-postgres) avec connection pooling, et de créer un script `init.sql` pour initialiser le schéma au démarrage du conteneur.
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

## Phase 2 : Conteneurisation avec Docker

### Dockerfile du Backend
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

### Dockerfile du Frontend (Multi-stage)

Le frontend utilise une approche de build multi-stage : la première image Node.js compile les assets avec Vite, la seconde image Nginx sert les fichiers compilés. L'image finale ne contient que Nginx et les fichiers statiques, sans aucun outil Node.js.
```dockerfile
# Étape 1 : Build avec Node.js
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Étape 2 : Serveur Nginx (seule cette étape est dans l'image finale)
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### docker-compose.yml

Le fichier docker-compose orchestre les trois services avec healthchecks et dépendances :
```yaml
services:
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

  backend:
    build: ./backend
    ports: ["3000:3000"]
    environment:
      DB_HOST: db
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build: ./frontend
    ports: ["5173:80"]
    depends_on:
      - backend
```

Le `depends_on` avec `condition: service_healthy` garantit que le backend ne démarre qu'une fois PostgreSQL opérationnel, évitant les erreurs de connexion au démarrage.

---

## Phase 3 : Pipeline CI/CD avec GitHub Actions

### Architecture du Pipeline

Deux pipelines GitHub Actions ont été mis en place. Le premier (`backend-ci.yml`) valide le code à chaque push et pull request. Le second (`ci.yml`) build et push les images Docker sur Docker Hub.

### Pipeline CI — Tests automatisés
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: tasksdb
        options: --health-cmd pg_isready --health-interval 10s
```

PostgreSQL est lancé directement comme service GitHub Actions, permettant aux tests Jest/Supertest de s'exécuter contre une vraie base de données.

### Pipeline CD — Build et Push Docker Hub
```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v2
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Build and push Backend
  uses: docker/build-push-action@v4
  with:
    context: ./Project/backend
    push: true
    tags: rsididris/task-manager-backend:latest

- name: Build and push Frontend
  uses: docker/build-push-action@v4
  with:
    context: ./Project/frontend
    push: true
    tags: rsididris/task-manager-frontend:latest
```

Les identifiants Docker Hub sont stockés dans les secrets GitHub (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`) et ne sont jamais visibles dans les logs.

---

## Phase 4 : Orchestration Kubernetes

### Philosophie de Configuration

Les manifestes Kubernetes définissent l'état désiré du système. Kubernetes se charge de maintenir cet état en permanence, en redémarrant les pods crashés et en gérant les mises à jour.

### Déploiement PostgreSQL avec Stockage Persistant

La gestion de la base de données nécessite de la persistance : si le pod redémarre, les données ne doivent pas être perdues. Kubernetes offre le PersistentVolumeClaim (PVC) pour répondre à ce besoin.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
```

Le script d'initialisation SQL est injecté via un **ConfigMap**, remplaçant le fichier `init.sql` local :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init-sql
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS tasks (
        id SERIAL PRIMARY KEY,
        matiere TEXT,
        title TEXT NOT NULL,
        description TEXT,
        priority TEXT DEFAULT 'Basse',
        status TEXT DEFAULT 'À faire'
    );
```

### Déploiement Backend
```yaml
containers:
- name: backend
  image: rsididris/task-manager-backend:latest
  ports:
  - containerPort: 3000
  env:
  - name: DB_HOST
    value: "db"
```

### Déploiement Frontend avec NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```

Le service de type `NodePort` expose le frontend sur le port 30001 du nœud Kubernetes, permettant l'accès depuis l'extérieur du cluster.

---

## Lancer le Projet
```bash
git clone https://github.com/yanis-nouili/devops_base.git
cd devops_base/Project
docker-compose up --build
```

- Frontend : http://localhost:5173
- Backend API : http://localhost:3000/tasks
- Health check : http://localhost:3000/health

---

## Conclusion

Ce projet nous a permis de parcourir l'intégralité du cycle de vie d'une application cloud-native. Nous avons conteneurisé chaque composant avec Docker, automatisé les tests et le déploiement via GitHub Actions, et orchestré l'ensemble avec Kubernetes. La migration de SQLite vers PostgreSQL, la mise en place du healthcheck et l'utilisation du ConfigMap pour l'initialisation de la base de données illustrent les bonnes pratiques DevOps appliquées à un projet concret.