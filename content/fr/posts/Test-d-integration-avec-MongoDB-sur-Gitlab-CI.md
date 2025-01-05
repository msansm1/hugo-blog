---
title: "Tests d'intégration avec MongoDB sur Gitlab-CI"
description: "Comment faire un test d'intégration avec MongoDB dans un pipeline Gitlab-CI."
# 1. To ensure Netlify triggers a build on our exampleSite instance, we need to change a file in the exampleSite directory.
theme_version: '2.8.2'
date: 2022-08-04
publishDate: 2022-08-04
cascade:
featured_image: '/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green.png'
---

Chez Slickteam notre plateforme CI-CD fonctionne avec Gitlab-CI, et plutôt bien !
Nous appréhendons de plus en plus les capacités de la CI projet après projet, et nous voulons le faire de la meilleure façon possible.
Pour un des mes projets j'avais écrit des tests d'intégration avec la base de données, et je voulais qu'ils soient exécutés automatiquement dans notre pipeline de CI.

Pour pouvoir le faire, j'ai lu et testé pas mal de choses.
Pour mon test j'ai besoin d'une base de données initialisée avec des collections vides, et un utilisateur avec les droits de lecture et écriture.

Tout d'abord je voulais l'avoir directement dans le pipeline sans s'appuyer sur autre image Docker que celle officielle de MongoDB.
Mais après plusieurs essais je n'ai pas trouvé comment le faire fonctionner selon mon idée, et l'étape du pipeline contenait trop de commandes pour que cela reste lisible et maintenable.

Après j'ai décidé de démarrer ma base MongoDB comme un service dans Gitlab-CI, pour que cela fonctionne correctement je devais avoir une image Docker avec un service MongoDB et une base de données initialisé comme je le souhaitais.

En premier je dois avoir une image Docker avec une base MongoDB préconfigurée.
Pour faire cela, j'ai créé un projet dédié.

Ce projet contient 3 fichiers : la configuration pour Gitlab-CI, un fichier Dockerfile et un script d'initialisation pour MongoDB.

Vous pouvez initialiser une base MongoDB au sein d'un conteneur grâce à un fichier javascript.
Ce fichier va contenir des instructions mongo shell pour configurer votre base de données.
Je l'utilise pour créer un utilisateur de la base, configurer ses droits et créer les collections.
Il ressemble à ça :
Init script for the database

J'ai un Dockerfile qui construit mon image avec ce script et configure l'utilisateur root de MongoDB :
Dockerfile for testing image

Pour compléter ceci, j'ai mon fichier .gitlab-ci.yml qui construit l'image et la pousse sur notre dépôt automatiquement :
.gitlab-ci file for testing image

Maintenant que j'ai mon image MongoDB configurée dans notre dépôt, je peux l'utiliser dans mon projet afin de lancer les tests d'intégration avec elle.

Dans Gitlab-CI, j'ai un pipeline complet pour construire mon projet. Une étape lance les tests d'intégration, et pour cela je vais utiliser ma nouvelle image Docker pour avoir un service MongoDB configuré comme je le souhaite.

Ci-dessous un exemple de test d'intégration que j'ai fait, notre projet est écrit en Kotlin (avec Javalin) et j'ai utilisé KMongo pour connecter ma base de données :
Integration test class

J'ai auusi configuré une étape de façon à ce que les résultats des tests soient affichés dans la page d'exécution du pipeline dans Gitlab-CI.
Stage in gitlab-ci.yml for integration tests

Dans cette étape j'ai configuré un alias pour mon service (mongo) et je l'utilise dans mon URI de connexion à MongoDB.

Maintenant si vous lancez votre pipeline, vos tests doivent être exécutés :
![Résultats des tests l'IHM de Gilab-CI](/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green.png "Résultats des tests l'IHM de Gilab-CI")

L'alias pour le service n'est pas connu localement, j'ai donc fait une petite modification dans ma configuration afin que les tests puissent fonctionner localement (dans l'IDE).

Pour avoir les tests qui fonctionnent en local, j'ai commencé par écrire un fichier docker-compose pour avoir MongoDB :
docker-compose file to run locally MongoDB

J'ai besoin d'avoir une configuration qui va mettre l'hôte Mongo en tant que localhost et non en tant que mongo comme pour la CI.
Pour avoir 2 configurations différentes j'ai mis une priorité dans la configuration de la librairie que j'utilise (Konfig) :
Priority in configurations

Du coup j'ai un fichier dev.properties dans le dossier local d'exécution de mon application qui surdéfini le fichier par défaut.
J'ai ajouté ce fichier à la racine de mon projet, et pour m'assurer qu'il ne soit pas commit je l'ai déclaré dans le fichier .gitignore.
Dans mon fichier dev.properties j'ai ajouté les mêmes propriétés que celles par défaut ainsi je peux gérer facilement si nécessaire :
MongoDB connection configuration

Et c'est tout ! Maintenant les tests peuvent être exécutés dans l'IDE :
![C'est vert !](/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green_ide.png "C'est vert !")

Vous pouvez trouver tus les fichiers et un projet fonctionnel dans ce dépôt github.
J'espère que cela vous aidera à tester votre application.

Si vous avez aimé cet article ou l'avez trouvé utile, n'hésitez pas à le partager ! 🙂
