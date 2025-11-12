---
title: "Migrer depuis SpringBoot 1.5 vers SpringBoot 2.5 avec Openrewrite"
description: "Comment migrer une application depuis SpringBoot 1.5 vers SpringBoot 2.5 avec Openrewrite."
date: 2022-08-04
publishDate: 2022-08-04
featuredImage: '/images/mng_springboot_15_25_migrate/logo.png'
code:
  copy: true
---

Chez [Slickteam](https://slickteam.fr/), nous avons un vieux projet client qui fonctionne avec [SpringBoot](https://spring.io/projects/spring-boot) 1.5 qui est en production.
Lorsque j'ai vérifié si nous avions des failles de sécurité avec la pile technique utilisée, j'ai constaté que la version de SprigBoot n'était plus maintenue depuis plusieurs années. Il y a plusieurs CVEs qui ne seront jamais corrigées.
Ainsi j'ai planifié la migration de l'application vers SpringBoot 2.5, à jour avec tous les correctifs de sécurité.

Par conséquent, j'ai cherché comment je pourrais le faire, et après plusieurs articles j'ai découvert [OpenRewrite](https://docs.openrewrite.org/), un projet destiné à faire des migrations.
J'ai décidé d'y jeter un coup d'œil, et de l'essayer pour faire ma migration.

OpenRewrite est un projet que vous pouvez utiliser à travers des plugins Maven ou Gradle.
Le plugin peut gérer plusieurs migrations, comme SpringBoot 1.x vers 2.x, JUnit 4 vers 5, Java 8 vers 11, …
J'ai décidé de l'utiliser pour migrer notre application de SpringBoot 1.5 vers SpringBoot 2.5, avec également Java 8 vers 11.

Notre projet utilise Gradle, j'ai donc utilisé le plugin Gradle.
J'ai suivi [ce guide](https://docs.openrewrite.org/tutorials/running-recipes/spring-boot-2.x-migration-from-spring-boot-1.x).
J'ai commencé par mettre à jour mon fichier gradle.build.kts come ceci :

```kotlin
plugins {
    ...
    id("org.openrewrite.rewrite") version "5.20.0"
}

dependencies {
    ...
    rewrite("org.openrewrite.recipe:rewrite-migrate-java:1.4.3")
    rewrite("org.openrewrite.recipe:rewrite-spring:4.19.3")
}

rewrite {
    activeRecipe("org.openrewrite.java.migrate.Java8toJava11")
    activeRecipe( "org.openrewrite.java.spring.boot2.SpringBoot1To2Migration")
}
```

J'ai ajouté le plugin Gradle rewrite, ses dépendances, et il est configuré pour activer la recette de migration SpringBoot.

Après sa configuration, vous pouvez lancer la migration :

```shell
# ./gradlew rewriteRun
```

Vous devriez voir ceci dans la sortie console :

```shell
> Task :rewriteRun
Validating active recipes
Parsing sources from project myproject
...
All sources parsed, running active recipes:
org.openrewrite.java.migrate.Java8toJava11,
org.openrewrite.java.spring.boot2.SpringBoot1To2Migration
Changes have been made to src/intTest/java/com/.../..Test.java by:
  org.openrewrite.java.spring.boot2.SpringBoot1To2Migration
    org.openrewrite.java.spring.boot2.UpgradeSpringBoot_2_5
```

Vous pouvez y voir tous les fichiers migrés, ainsi que ceux ayant échoués à cause d'une erreur. Lorsque je l'ai fait la première fois, il m'a semblé que les versions des dépendances dans le fichier build.gradle.kts avaient été correctement mises à jour (il y a 8 mois), mais lors de mon dernier essai le fichier n'a pas du tout été mis à jour. Peut-être ai-je manqué quelque chose. Après cette exécution, vous pouvez vérifier avec un diff les changements dans vos fichiers, et corriger ou directement commit si tout est bon.

Quand je l'ai fait sur le projet j'ai été impressionné par l'outil, il est vraiment simple d'usage et les résultats très bon. Cela a pris une demie journée pour effectuer la première migration, puis une journée de corrections et ajustements du fait de certaines dépendances du projet (comme spring social). J'ai eu quelques soucis avec Thymeleaf, mais je pense que le problème venait plus de la façon dont le projet a été codé initialement que de la migration.

Si vous avez une migration à faire, et qu'un recette OpenRewrite existe, je vous recommanderai d'y jeter un coup d'oeil et de l'essayer. Vous allez probablement gagner du temps et éviter des prises de têtes !

Si vous avez aimé ou trouvé cet article utile, n'hésitez pas à le partager ! 🙂
