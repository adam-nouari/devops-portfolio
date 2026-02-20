---
title: Lab 3 - Déploiement d'Applications
---

# Lab 3 - Déploiement d'Applications

## Objectifs
Explorer différentes méthodes d'orchestration pour déployer des applications.

## Partie 1 : Orchestration Serveur avec Ansible
Déploiement de 3 instances EC2 avec Node.js et mise en place d'un load balancer Nginx. Implémentation de rolling updates avec `serial: 1`.

## Partie 2 : Orchestration VM avec Packer et OpenTofu
Création d'une AMI avec Packer, déploiement via Auto Scaling Group (ASG) et Application Load Balancer (ALB). Rolling updates avec ASG Instance Refresh.

## Partie 3 : Orchestration Container avec Docker et Kubernetes
Construction d'une image Docker, déploiement sur Kubernetes local puis sur EKS (AWS). Push de l'image sur Amazon ECR. Rolling updates Kubernetes.

## Partie 4 : Serverless avec AWS Lambda
Déploiement d'une fonction Lambda avec API Gateway via OpenTofu. Mise à jour rapide sans gestion de serveur.