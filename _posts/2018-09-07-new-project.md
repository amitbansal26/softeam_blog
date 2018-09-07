---
title: Mise en place d'un nouveau Projet dans l'Usine
date: 2018-09-07 00:00:00 Z
layout: post
author: Mehdi EL KOUHEN
description: Mise en place d'un nouveau Projet dans l'Usine
toc: false
---

L'objectif de cet article est de donner les étapes de mise en place d'une Application dans notre Usine.

## Construction & Déploiement des Applications 

Les sources de nos projets sont gérés dans l'organisation GitHub [SofteamOuest](https://github.com/SofteamOuest/).

Notre [Jenkins](jenkins.k8.wildwidewest.xyz) se synchronise automatiquement avec cette organisation.

* Dès qu'un dépôt est créé sur GitHub, Jenkins crée un Job pour construire et déployer l'Application (Définition du Job basé sur le Jenkinsfile s'il existe)

Le Job jenkins :

* Analyse la qualité du code de l'application (via [sonarqube](https://sonarqube.k8.wildwidewest.xyz/))
* Construit les images Docker de l'Application
* Déploie les images Docker sur [nexus](https://nexus.k8.wildwidewest.xyz/)

La fin d'exécution du Job Jenkins déclenche le déploiement de l'Application (par exécution du Job [chart-run](https://jenkins.k8.wildwidewest.xyz/job/SofteamOuest/job/chart-run/)) sur le cluster.

## Mise en place d'une Application

Pour mettre en place le déploiement sur le cluster, il faut :

* Intégrer au Projet, un Dockerfile pour construire l'image (ou les images de l'Application)
* Intégrer au Projet, un docker-compose.yml pour simplifier le démarrage de l'Application sur le Poste de Dev.
* Créer un chart [helm](https://helm.sh/) pour déployer l'Application dans le cluster.

## Création du Projet

Convention de nommage des Projet :

* Si projet FRONT, le nom est *domaine_fonctionnel* suffixé de _ui
* Si projet BACK, le nom est *domaine_fonctionnel* suffixé de _api

Le nom du domaine fonctionnel peut être par exemple le nom de la ressource gérée (exemple : users).

Remarque : Ne pas préfixer les noms des applications avec des mots génériques comme "gestion".

### Création du Dockerfile

Le Dockerfile permet de construire l'image Docker de l'application.

Exemple de Dockerfile pour une application java.

```bash
FROM java:9

MAINTAINER XXX@softeam.fr

WORKDIR /apps/monappli

COPY target/monappli.jar /apps/monappli/monappli.jar

EXPOSE 8080

CMD java -jar monappli.jar 
```

### Création du docker-compose.yml

Le docker-compose.yml simplifie la gestion des services docker. Il est aussi utilisé par le Jenkinsfile pour simplifier la construction des images Docker et leur *push* sur nexus.

Pour construire les images Docker (de l'Application) :

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

Exemple de Dockerfile pour une application java avec une base de données PostgreSQL.

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

### Création du package helm

Les charts Helm du Projet sont gérés dans le dépôt GIT [charts](https://github.com/SofteamOuest/charts).

Il faut d'abord installer [Helm](https://docs.helm.sh/using_helm/#installing-helm) sur le Poste de Développement.

Puis, pour créer un chart helm il suffit d'exécuter à la racine du dépôt [charts](https://github.com/SofteamOuest/charts).

```bash
helm create monappli
```

Et finalement modifier les fichiers générés (cf. sections ci-dessous).

#### Fichier values.yaml

Liste des Modifications :

* Le nom de l'image docker 

  * Le même nom que celui utilisé dans le docker-compose.yml
* La conf SSL de l'Application (si Application accessible en dehors du Cluster)

  * **ingress.enabled = true** => Création d'une ressource ingress pour que l'Application soit accessible sur internet
  * **ingress.hosts** => URL de l'Application
  * **ingress.secretName** => Nom du secret Kubernetes contenant les certificats de l'Application
* Les ressources allouées au conteneur déployé (à adapter à l'Application)

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

#### Fichier deployment.yaml

Liste des modifications :

* Intégrer le nom du secret Kubernetes (regsecret)contenant le login/mot de Nexus (nécessaire pour le download des images Docker)  

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

### Release du package helm

Pour qu'une version d'un package helm soit *visible* du cluster, il faut construire une version du package et le publier dans le repo helm.

La release n'est pas encore industrialisée.

Il faut vérifier la qualité des packages helm.

```bash
sh ./lint.sh
```

Il faut modifier la version du package a releaser (cf. fichier release.sh) 

```bash
version=0.1.43
```

Décommenter les packages à releaser.

```bash
helm package --version $version monappli
```

Effectuer la release.

```bash
sh ./release.sh
```

### Création du Jenkinsfile

La définition du Job Jenkins se fait via un Jenkinsfile (remplacer *monappli* par le vrai nom de l'Application).

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

## Vérification du Déploiement

Pour vérifier l'état du déploiement, il faut vérifier l'état du Job [chart-run](https://jenkins.k8.wildwidewest.xyz/job/SofteamOuest/job/chart-run/). 

Ensuite, il faut se connecter en SSH sur le master (du cluster) pour vérifier l'état du Pod "monappli" (qui est déployée dans le namespace dev)

```bash
[root@vps242131 ~]# kubectl get pod -l app=monappli --namespace dev
NAME                           READY     STATUS    RESTARTS   AGE
monappli-7bf958b897-ts9fc   1/1       Running   0          2h
```

Si l'option *ingress.enabled* est définie à true dans le package helm, l'application doit être accessible par internet. 

L'URL d'accès est le nom de l'application suivi du nom de l'environnement (ici dev) suivi de k8.wildwidewest.xyz.

* L'URL de l'appli est https://monappli-dev.k8.wildwidewest.xyz

Pour se connecter en SSH au cluster, il faut fournir à Mehdi une clef SSH publique pour qu'il l'enregistre dans les clefs autorisées par l'Usine.