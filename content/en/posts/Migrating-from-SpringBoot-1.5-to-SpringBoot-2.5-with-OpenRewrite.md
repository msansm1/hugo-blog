---
title: "Migrating from SpringBoot 1.5 to SpringBoot 2.5 with OpenRewrite"
description: "How to migrate from SpringBoot 1.5 to SpringBoot 2.5 with OpenRewrite."
date: 2022-08-04
publishDate: 2022-08-04
featuredImage: '/images/mng_springboot_15_25_migrate/logo.png'
code:
  copy: true
---

At [Slickteam](https://slickteam.fr/), we got an old project working with [SpringBoot](https://spring.io/projects/spring-boot) 1.5 running in production.
When I checked if we had security issues with this technical stack, I noticed that the version of SpringBoot we used hadn’t been maintained for a few years, there also are some CVEs that will never be corrected.
So I planned to migrate the application to SpringBoot 2.5, up to date with all security fixes.

Therefore, I searched how I could do it, and after few articles I discovered [OpenRewrite](https://docs.openrewrite.org/), a project that could help doing migrations.
I decided to take a good look at it, and to try it for my migration.

OpenRewrite is a project that you can use through a Maven or Gradle plugin.
The plugin can manage a few migrations, like SpringBoot 1.x to 2.x, JUnit 4 to 5, Java 8 to 11, …
I decided to use it to migrate our application from SpringBoot 1.5 to 2.5, along with Java 8 to 11.

Our project uses Gradle, so I used the Gradle plugin.
I followed [this guide](https://docs.openrewrite.org/tutorials/running-recipes/spring-boot-2.x-migration-from-spring-boot-1.x).
I started by updating my gradle.build.kts file, like this:

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

I added the rewrite Gradle plugin, the rewrite dependencies, and configured it to activate the recipe for SpringBoot migration.

After configuring it, you can run the migration:

```shell
# ./gradlew rewriteRun
```

You should see this in the output console:

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

In the output you will see all the files migrated, and the files where the migration failed due to an error.
When I did it the first time, it seemed to me that the dependencies versions in the build.gradle.kts were correctly updated (8 months ago), but in my last try the file was not updated at all.
Maybe I missed something.
After running it, you can check with a diff what are the changes in your files, and fix or directly commit if all is good.

When I did it on the project I was impressed by the tool, it is really simple to use and the results were really good. It took half a day to do the first migration, then 1 day for fixing some specific things we used (like spring social). I got some issues with Thymeleaf, but I think the problem is more about the way the development was initially done rather than the migration.

If you have a migration to do, and a recipe exists in OpenRewrite, I would recommend you to check it, and try it out.
You probably will save much time and avoid headaches !

If you liked this article or you found it useful, don’t hesitate to 👏 it or to share it ! 🙂
