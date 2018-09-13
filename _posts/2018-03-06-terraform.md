---
title: Terraform
date: '2018-03-06 00:00:00 Z'
layout: post
description: Terraform
img: software-factory.jpg
author: Victor SALAUN
---

[Terraform](https://www.terraform.io/) est un outil d’infrastructure as code, multi-providers, open source, édité par [Hashicorp](https://www.hashicorp.com/).
Il permet de créer et modifier ses infrastructures et environements sous la forme de fichiers de configuration de manière prévisible.
Le principal concept se base sur l’état de la cible, il s’agit donc d’une approche d’infrastructure immutable.


Avantages :
* Est devenu un standard
* Large communauté
* Support multi-provides assuré dont AWS, Azure, GCE
* Reproductibilité via des modules
* Extensibilité par plugin


Inconvénients :
* Pas d’interface centralisée
* Nécessité de développer
* En fonction du provider, pas tous les services ne sont couverts

La création de modules permet de centraliser les usages et d’assurer une bonne réutilisabilité des modèles d’infrastructures par les équipes

# Gérer son organisation GitHub avec Terraform

## Structure

Il existe une convention structurant les dépôt Terraform.

    ├── README.md
    ├── variables.tf
    ├── main.tf
    ├── outputs.tf
    ├── modules
    │   ├── compute
    │   │   ├── README.md
    │   │   ├── variables.tf
    │   │   ├── main.tf
    │   │   ├── outputs.tf
    │   ├── networking
    │   │   ├── README.md
    │   │   ├── variables.tf
    │   │   ├── main.tf
    │   │   ├── outputs.tf

Les fichiers de configuration à retenir sont :
* `variables.tf` permettant de stocker des variables tels que des clefs, secrets ou valeurs par défaut.
* `main.tf` contient les définitions des resources et modules
* `outputs.tf` les données de sorties

## GitHub

Prenons l'exemple simple la gestion d'un utilisateur au sein de son organisation Github:

`variables.tf`

    variable "github_token" {}
    
    variable "github_organization" {
      default = "SofteamOuest"
    }
    
Il n'est pas recommandé de publier son token GitHub sur le dépôt, il faut alors donner le token à chaque commande Terraform :

    apply -input=false -var "github_token=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    

`main.tf`

    data "github_user" "victorsalaun" {
      username = "victorsalaun"
      role = "admin"
    }

Une fois les fichiers de configuration appliqués, nous obtenons un nouveau fichier `terraform.tfstate` qui permet à Terraform de connaître l'état existant avant modification.
D'ordinaire il n'est pas recommandé de publier ce fichier sur le dépôt car il peut contenir des clefs selon le provider. Concernant GitHub nous n'en avons pas, sinon une solution proposée serait de le publier sur S3 de AWS.

## Application de la configuration

A la première exécution, il faut lancer la commande suivante afin que terraform récupère le plugin GitHub :

    terraform init
    
Ensuite il est possible de connaître les changements avant des les appliquer :

    terraform plan

Pour finir, pour indiquer à Terraform d'appliquer les modifications :

    terraform apply