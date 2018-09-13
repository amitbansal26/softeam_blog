---
title: Ma première journée avec Vue.js
date: '2018-01-19 00:00:00 Z'
layout: post
description: Tutorial VueJS
img: software-factory.jpg
author: Mehdi El Kouhen
---

L'objectif de cet article est de présenter un retour d'expérience d'une première utilisation de [Vue.js](https://vuejs.org/).

# Description de l'application

L'application contient deux pages :

* Une page d'affichage de tous les événements dans un tableau.
* Une page d'édition d'un événement donné.

Pour le développement de l'application, j'ai développé 3 composants : 

* un composant EventList d'affichage d'une liste d'événements sous forme tabulaire
* un composant Event d'affichage d'un événement 
* un composant EventBadge d'affichage du statut d'un événement

J'ai intégré bootstrap pour le style CSS, le routeur par défaut du Vue.js (pour le routage entre les pages) et axios pour les requêtes Ajax.

# Composant

## Structure d'un composant

Vue.js propose de structurer le code d'un composant dans un unique fichier (suffixé .vue).

```
<template>
# code HTML
</template>

<script>
# code JS
</script>

<style>
# code CSS
</style>
```

Ceci permet de colocaliser le code HTML/JS/CSS d'un composant dans un fichier unique. Pour ma part, j'ai préféré conserver le code HTML/JS/CSS dans des fichiers dédiés et les importer dans les composants Vue (cf. attribut src).

```
<template src='./Event.html'>

</template>

<script src='./Event.js'>

</script>

<style scoped>

</style>
```

## Structure JS d'un composant

Le fragment JS du [composant](https://fr.vuejs.org/v2/guide/components.html) contient

* des attributs (exemple : name -> nom du composant)
* des méthodes (exemple : la méthode created est appelée à la création du composant)

```javascript
export default {
  name: 'EventList',
  data() {
    return {
      items: []
    }
  },
  created() {
    // appel ajax
    axios.get(eventURL).then(response => { this.items = response.data; });
  }
}
```

## Initialisation des données d'un composant

Le composant EventList est responsable de télécharger tous les événements avant de les afficher.

* la fonction 'data' du composant contient les données du composant
* la fonction 'created' est exécutée à la création du composant (téléchargement des événements et stockage dans data)

```javascript
export default {
  name: 'EventList',
  data() {
    return {
      items: []
    }
  },
  created() {
    // appel ajax
    axios.get(eventURL).then(response => { this.items = response.data; });
  }
}
```

## Paramétrage d'un composant

Les composants Vue.js sont paramétrables via des [props](https://fr.vuejs.org/v2/guide/components.html#Props). Le composant ci-dessous à un props : le status.

```javascript
export default {
  name: 'event-badge',
  props: ['status'],
  ...
}
```

A l'utilisation du composant, il faut valoriser les props.

```
<event-badge :status='props.item.status'>{{ props.item.status }}</event-badge>
```

# Intégration de composants externes

## Intégration de Bootstrap

Pour intégrer Bootstrap dans le code, je me suis servi du projet [Bootstrap + Vue](https://bootstrap-vue.js.org/). Le projet [Bootstrap + Vue](https://bootstrap-vue.js.org/) apporte des composants Vue.js pour chaque composant Bootstrap.

Exemple de barre de navigation avec Bootstrap + Vue. 

```html
<b-navbar toggleable type="light" variant="light">
    <b-navbar-toggle target="nav_text_collapse"></b-navbar-toggle>
    <b-navbar-brand>EventMgr</b-navbar-brand>
    <b-collapse is-nav id="nav_text_collapse">
      <b-navbar-nav>
        <b-nav-item href="#/events">Home</b-nav-item>
        <b-nav-item href="#/events/1">Events</b-nav-item>
      </b-navbar-nav>
    </b-collapse>
</b-navbar>
```

Comme décrit [dans la documentation du projet](https://bootstrap-vue.js.org/docs), il suffit d'ajouter au projet les dépendances bootstrap-vue et bootstrap puis d'enregistrer le plugin BootstrapVue dans le composant "point d'entrée" (composant App.vue).

```javascript
import BootstrapVue from 'bootstrap-vue'

// nécessaire pour webpack
import 'bootstrap/dist/css/bootstrap.css'
import 'bootstrap-vue/dist/bootstrap-vue.css

Vue.use(BootstrapVue);
```

## Intégration d'un routeur

Le projet généré intègre le [routeur par défaut](https://github.com/vuejs/vue-router) de Vue.js.

Le routeur est paramétré par un ensemble de routes :

* une route /events (pour afficher tous les événements)
* une route paramétrée /events/:id par l'identifiant d'un événement (pour afficher l'événement)

```javascript
export default new Router({
  routes: [
    {
      path: '/events/:id',
      name: 'Event',
      component: Event
    }, {
      path: '/events',
      name: 'EventList',
      component: EventList
    }
  ]
})
```

L'intégration du routeur se fait au niveau du composant App.vue (composant point d'entrée) par intégration du fragment HTML ci-dessous (dans le code HTML du composant).

```
 <router-view/>
```

## Intégration d'un client HTTP

En suivant le tutorial [alligator.io](https://alligator.io/vuejs/rest-api-axios/), j'ai intégré le client axios.

# Le Projet

Les sources du projet sont disponibles sur [github](https://github.com/SofteamOuest/event-gui).

J'ai utilisé [JSON Server](https://github.com/typicode/json-server) pour mettre en place rapidement une API de gestion d'événements. La base de données JSON est disponible sur [github](https://github.com/SofteamOuest/event-api-mock).

## Génération du projet

Pour générer la structure du projet, j'ai utilisé [vue-cli](https://github.com/vuejs/vue-cli).

## Compilation du projet 

La compilation du projet est géré par [webpack](https://webpack.js.org/).

### Compilation Live Reload

La compilation à chaud se fait en exécutant la ligne de commande suivante : webpack publie les sources du projet (via un serveur http local) sur http://localhost:8080

```bash
npm run dev
```

### Génération des fichiers statiques

La génération des fichiers HTML/JS/CSS se fait en exécutant la ligne de commande suivante :

```bash
npm run build
```