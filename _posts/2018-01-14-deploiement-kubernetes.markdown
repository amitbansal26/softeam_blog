---
title: Déploiement d'une application dans Kubernetes
date: '2018-01-14 00:00:00 Z'
layout: post
description: Tutorial Kubernetes
img: software-factory.jpg
author: Mehdi El Kouhen
---

L'objectif de cet article est de présenter les concepts de base nécessaires pour le déploiement d'une application Dockerisée dans un cluster Kubernetes.

# Dépôts GIT 

Le code présenté dans cet article est extrait des deux dépôts GIT suivants.

## [books-api-simple](https://github.com/SofteamOuest/books-api-simple)

API de gestion de livres.

## [books-api-simple-run](https://github.com/SofteamOuest/books-api-simple-run)

Module de déploiement de l'API dans Kubernetes.

# Déploiement de l'API

Le déploiement du conteneur de l'API est réalisé en définissant un objet [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Le résultat du déploiement est la création d'un [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) dans le cluster.

Le déployment spécifie :

* l'image Docker à déployer (registry.k8.wildwidewest.xyz/repository/docker-repository/pocs/books-api-simple)
* le nombre de replicas (1 ci-dessous))
* le port exposé par l'application (8080 ci-dessous)
* le mot de passe d'accès au registry docker stocké sous forme de secret (regsecret ci-dessous)

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: books-api-simple
spec:
  replicas: 1
  template:
    metadata:
     labels:
        app: books-api-simple
    spec:
      containers:
      - name: books-api-simple
        image: registry.k8.wildwidewest.xyz/repository/docker-repository/pocs/books-api-simple:version
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: regsecret 

```

## Création d'un service d'accès

Dans Kubernetes, les Pods étant des objets éphémères, le [Service](https://kubernetes.io/docs/concepts/services-networking/service/) est le point d'accès, interne au cluster, aux conteneurs déployés.

Kubernetes enregistre les Services dans le DNS local au cluster, l'appel d'un service se fait donc via son nom.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: books-api-simple
spec:
  selector:
    app: books-api-simple
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

## Publication du service 

Kubernetes permet via la notion d'[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) de publier les services en dehors du cluster.

Le fragment de code ci-dessous map les requêtes envoyées à l'URL books-api-simple.k8.wildwidewest.xyz vers le service books-api-simple.

Le proxy utilisé est un proxy [traefik](https://traefik.io/).

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: books-api-simple
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: books-api-simple.k8.wildwidewest.xyz
    http:
      paths:
      - path: /
        backend:
          serviceName: books-api-simple
          servicePort: 80
  tls:
  - secretName: traefik-cert
```
