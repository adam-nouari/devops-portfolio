---
title: Projet Final - Task Manager
---

# Projet Final - Task Manager

## Description
Application de gestion de tâches développée en groupe avec une architecture microservices conteneurisée.

## Stack Technique
- **Frontend** : React + Vite, servi par Nginx
- **Backend** : Node.js + Express + pg
- **Base de données** : PostgreSQL 15
- **Conteneurisation** : Docker + docker-compose
- **CI/CD** : GitHub Actions
- **Registry** : Docker Hub

## Architecture
3 conteneurs orchestrés par docker-compose avec healthchecks et dépendances entre services.

## Base de données (ma partie)
Migration de SQLite vers PostgreSQL avec volume persistant. Script d'initialisation `init.sql` converti en ConfigMap Kubernetes. CRUD complet validé via tests Jest/Supertest dans le pipeline CI.

## CI/CD Pipeline
- Build et tests automatiques à chaque push
- Service PostgreSQL intégré dans GitHub Actions
- Push automatique sur Docker Hub

## Kubernetes
Manifestes de déploiement pour les 3 services avec PersistentVolumeClaim pour PostgreSQL.

## Repo GitHub
[yanis-nouili/devops_base](https://github.com/yanis-nouili/devops_base)