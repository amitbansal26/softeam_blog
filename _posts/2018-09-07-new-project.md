---
title: Mise en place d'un nouveau Projet dans l'Usine
date: 2018-09-07
layout: post
author: Mehdi EL KOUHEN
description: Mise en place d'un nouveau Projet dans l'Usine
excerpt_separator: "<!--more-->"
---

## Démarche

L'environnement de déploiement est un cluster Kubernetes.

Il faut donc :

* Choisir un nom adéquat au projet 
* Créer un Dockerfile pour construire l'image (ou les images de l'application)
* Créer un docker-compose.yml pour simplifier le démarrage de l'application sur le Poste de Dev.
* Créer un chart [helm](https://helm.sh/) pour déployer l'application dans le cluster.

## Création du Projet

Les sources de nos projets sont gérés dans l'organisation GITHUB [SofteamOuest](https://github.com/SofteamOuest/).

Convention de nommage des Projet : 

* Si projet FRONT, le nom est *domaine_fonctionnel* suffixé de _ui 
* Si projet BACK, le nom est *domaine_fonctionnel* suffixé de _api

## Création du Dockerfile

Exemple de Dockerfile pour une application java 

```
FROM java:9

MAINTAINER XXX@softeam.fr

WORKDIR /apps/monappli

COPY target/monappli.jar /apps/monappli/monappli.jar

EXPOSE 8080

CMD java -jar monappli.jar 
```

## Création du docker-compose.yml

Le docker-compose.yml simplifie la gestion des services docker.

Pour construire les images Docker (de l'application) :

```bash
docker-compose build
```

Pour démarrer les services :
```bash
docker-compose up
```

Pour arrêter les services :
```bash
docker-compose stop
```

Exemple de Dockerfile pour une application java avec une base de données PostgreSQL

```yaml
version: '3.2'
services:

  monappli-postgres:
    image: registry.k8.wildwidewest.xyz/repository/docker-repository/monappli-postgres:${tag}
    build:
      context: .
      dockerfile: Dockerfile-postgres

  monappli:
    image: registry.k8.wildwidewest.xyz/repository/docker-repository/monappli:${tag}
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
    - monappli-postgres
    ports:
    - "8080:8080"
```

## Création du package helm

Les charts Helm du Projet sont gérés dans le dépôt GIT [charts](https://github.com/SofteamOuest/charts).

Il faut installer [Helm](https://docs.helm.sh/using_helm/#installing-helm) sur le Poste de Développement.

Pour créer un chart helm il suffit d'exécuter à la racine du dépôt.

```bash
helm create monappli
```

### Fichier values.yaml

Liste des Modifications :

* Le nom de l'image docker 

  * Le même nom que celui utilisé dans le docker-compose.yml
* La conf SSL de l'Application (si Application accessible en dehors du Cluster)

  * **ingress.enabled = true** => Création d'une ressource ingress pour que l'application soit accessible sur internet
  * **ingress.hosts** => URL de l'application
  * **ingress.secretName** => Nom du secret Kubernetes contenant les certificats de l'application
* Les ressources allouées au conteneur déployé (à adapter à l'application)

```yaml
image:
  repository: registry.k8.wildwidewest.xyz/repository/docker-repository/monappli

ingress:
  enabled: true
  annotations:
    certmanager.k8s.io/cluster-issuer: letsencrypt-issuer
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
  path: /
  hosts:
  - monappli.k8.wildwidewest.xyz
  secretName: certificate
....
resources:
  limits:
    #  cpu: 100m
    memory: 128Mi
  requests:
    #  cpu: 100m
    memory: 64Mi
...  
```

### Fichier deployment.yaml

Liste des modifications :

* Intégrer le nom du secret Kubernetes contenant le login/mot de Nexus (pour le download des images Docker du cluster)  

```yaml
    spec:
      containers:
        - name: {{ .Chart.Name }}
...
      tolerations:
{{ toYaml . | indent 8 }}
    {{- end }}
      imagePullSecrets:
      - name: regsecret
```

## Création du Jenkinsfile

La définition du Job Jenkins se fait via un Jenkinsfile (remplacer *monappli* par le vrai nom de l'application).

```groovy
#!groovy
import java.text.*

// pod utilisé pour la compilation du projet
podTemplate(label: 'monappli-pod', containers: [

        // le slave jenkins
        containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:alpine'),

        // un conteneur pour le build maven
        containerTemplate(name: 'maven', image: 'maven', privileged: true, ttyEnabled: true, command: 'cat'),

        // un conteneur pour construire les images docker
        containerTemplate(name: 'docker', image: 'tmaier/docker-compose', command: 'cat', ttyEnabled: true),

        // un conteneur pour déployer les services kubernetes
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', command: 'cat', ttyEnabled: true)],

        // montage nécessaire pour que le conteneur docker fonction (Docker In Docker)
        volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
) {

    node('monappli-pod') {        

        properties([
                buildDiscarder(
                        logRotator(
                                artifactDaysToKeepStr: '1',
                                artifactNumToKeepStr: '1',
                                daysToKeepStr: '3',
                                numToKeepStr: '3'
                        )
                )
            ])

        def now = new SimpleDateFormat("yyyyMMddHHmmss").format(new Date())

        stage('checkout sources') {
            checkout scm
        }

        container('maven') {

            stage('build sources') {
                sh 'mvn clean install -DskipTests sonar:sonar -Dsonar.host.url=http://sonarqube-sonarqube:9000 -Dsonar.java.binaries=target'
            }
        }

        container('docker') {

            stage('build docker image') {

                    sh 'mkdir /etc/docker'

                    sh 'echo {"insecure-registries" : ["registry.k8.wildwidewest.xyz"]} > /etc/docker/daemon.json'

                    withCredentials([usernamePassword(credentialsId: 'nexus_user', usernameVariable: 'username', passwordVariable: 'password')]) {

                         sh "docker login -u ${username} -p ${password} registry.k8.wildwidewest.xyz"
                    }

                    sh "tag=$now docker-compose build"

                    sh "tag=$now docker-compose push"
            }
        }

        container('kubectl') {

            stage('deploy') {


                // Déclencher le déploiement 
                build job: '/SOFTEAMOUEST/chart-run/master', parameters: [
                    string(name: 'image', value: "$now"),
                    string(name: 'chart', value: "monappli")], wait: false
            }
        }
    }
}

```