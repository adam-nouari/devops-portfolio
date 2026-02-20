---
title: Lab 5 - CI/CD avec Kubernetes
---

# Lab 5 - CI/CD avec Kubernetes

## Objectifs
Mettre en place un pipeline CI/CD complet avec GitHub Actions et Kubernetes.

## Partie 1 : Intégration Continue (CI)
Pipeline GitHub Actions pour builder, tester et valider le code à chaque push. Tests automatisés avec PostgreSQL comme service dans le pipeline.

## Partie 2 : Livraison Continue (CD)
Build et push automatique des images Docker sur Docker Hub. Déploiement continu vers un cluster Kubernetes.

## Partie 3 : Déploiement Kubernetes
Manifestes Kubernetes pour le frontend, backend et PostgreSQL. PersistentVolumeClaim pour la persistence des données. ConfigMap pour l'initialisation de la base de données.