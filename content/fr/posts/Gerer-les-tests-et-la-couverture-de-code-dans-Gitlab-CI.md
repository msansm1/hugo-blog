---
title: "Gérer les tests et la couverture de code dans Gitlab-CI"
description: "Comment gérer les tests et afficher la couverture de code dans Gitlab-CI."
# 1. To ensure Netlify triggers a build on our exampleSite instance, we need to change a file in the exampleSite directory.
theme_version: '2.8.2'
date: 2021-05-11
publishDate: 2021-05-11
cascade:
featured_image: '/images/manage_test_coverage/manage_test_coverage_Pipeline_results.png'
---

Lorsque vous avez une CI, vous voulez qu'elle exécute les tests et affiche les résultats, ainsi que ce qui a échoué si c'est le cas.
Chez Slickteam, notre CI fonctionne avec Gitlab-CI, et nous gérons nos tests avec.
Cela nous aide à trouver plus facilement le problème en cas d'échec.
Nous affichons également le taux de couverture global pour tous les tests.

Voici comment nous le faisons.

Pour commencer vous devez avoir des tests dans votre code (qui n'en a pas ?) et les exécuter avec un outil de build ou autre.
Nous utilisons Grad le principalement, donc tous mes exemples vont utiliser des commandes Gradle.

J'ai créé des tests unitaires et des tests d'intégration dans le projet d'exemple, et divisé l'exécution des tests en 2 étapes différentes dans la configuration du pipeline :
Stages configuration in gitlab-ci.yml file

Pour chaque étape nous avons une commande d'exécution des tests (script), si nous voulons autoriser des échecs et obtenir les artefacts.
Ici j'ai autorisé les tests unitaires à échouer comme je veux que les 2 étapes soient jouées à chaque exécution du pipeline.

La partie 'artifact' sert à dire à Gitlab-CI où se trouvent vos rapports, et comment les gérer.
J'ai configuré "when: always" comme je souhaite que mes rapports soient tout le temps sauvegardés, et ainsi les avoir dans l'IHM pour chaque exécution de pipeline.
Gitlab-CI supporte le format des rapports JUnit, donc rien de plus à configurer ici.

Dans l'étape IT_test j'utilise un service, vous pouvez voir dans cet article pourquoi j'en ai besoin pour mes tests d'intégration.

Désormais, si vous avez un test en échec, vous verrez les résultats de cette façon dans l'interface :
![Interface des résultats de tests Gitlab-CI avec des tests en échec et la stacktrace de l'erreur](/images/manage_test_coverage/manage_test_coverage_Failed_test_UI.png "Résultat des tests en échec dans l'interface, avec la stacktrace.")

Comme ça vous pouvez voir la stack d'erreur directement dans l'interface, et ainsi trouver plus facilement l'origine du problème.

Lorsque vous commencez à avoir un nombre important de tests dans votre projet, vous pouvez vouloir connaître le taux de couverture.
Ainsi vous pouvez contrôler si votre code est bien couvert ou si certaines parties du code manquent de tests.
Nous utilisons JaCoCo dans nos projets Java pour mesurer la couverture, et nous avons configuré notre CI pour afficher le taux de couverture global dans l'interface.

Premièrement, vous devez récupérer les résultats depuis votre rapport de tests.
Comme nous avons 2 étapes pour les tests, nous voulons avoir le taux de couverture global avec les résultats des tests unitaires et d'intégration agrégés.
Dans le fichier gitlab-ci.yml, nous avons configuré les dossiers des rapports Jacoco comme artefact pour les 2 étapes, et nous avons déclaré l'étape unit_test comme dépendance de IT_test.
Ainsi, IT_test va récupérer les artefacts de l'étape unit_test, et JaCoCo sera capable de calculer le taux de couverture global pour tous les tests.

Vous pouvez voir que nous avons renommé 2 fichiers dans la partie script de l'étape, afin d'éviter que JaCoCo n'écrase le résultat de l'étape précédente.

Maintenant il faut configurer le format des résultats des tests pour la couverture dans Gitlab-CI.
Dans Paramètres -> CI/CD, à la section Général pipelines, vous pouvez configurer ce format ici :
![Page de paramétrage de Gitlab-CI pour configurer le format du taux de couverture](/images/manage_test_coverage/manage_test_coverage_Result_format_UI.png "Gérer le format de vos résultats de couverture dans l'interface.")

Pour notre projet, avec JaCoCo, nous avons utilisé cette expression régulière :

```js
Total.*?([0–9]{1,3})%
```

Avec tout ceci configuré, vous avez quelque chose comme ça dans l'interface :
![Page de résultat du pipeline de Gitlab-CI montrant le résultat de couverture pour la dernière étape](/images/manage_test_coverage/manage_test_coverage_Pipeline_results.png "Résultat du pipeline dans Gitlab-CI montrant le résultat de couverture pour la dernière étape.")

Si vous récupérez votre rapport JaCoCo dans les artefacts de l'étape, il montrera le même résultat :
![Rapport HTML de l'exécution de JaCoCo montrant le même résultat que dans Gitlab-CI](/images/manage_test_coverage/manage_test_coverage_Result_report.png "Rapport HTML de l'exécution de JaCoCo montrant le même résultat que dans Gitlab-CI.")

J'espère que tout ceci vous aidera à avoir de meilleures informations sur vos exécutions de pipeline dans Gitlab-CI, afin d'améliorer encore plus vos logiciels :).

Vous pouvez trouver un projet d'exemple sur Github ou Gitlab.
