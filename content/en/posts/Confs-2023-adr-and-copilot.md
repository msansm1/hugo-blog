---
title: "Retours de confs 2023 : ADR et Copilot"
description: "Retours de conférences 2023 : ADR au Breizhcamp et Copilot au Devfest Perros-Guirec."
# 1. To ensure Netlify triggers a build on our exampleSite instance, we need to change a file in the exampleSite directory.
theme_version: '2.8.2'
date: 2024-03-07
publishDate: 2024-03-07
cascade:
featured_image: '/images/mng_springboot_15_25_migrate/logo.png'
---


Cette année j’ai eu l’occasion d’assister à 2 conférences : le [Breizhcamp](https://www.breizhcamp.org/) à Rennes et le [DevFest Perros-Guirec](https://devfest.codedarmor.fr/). Parmi toutes les conférences intéressantes que j’ai vu, je voulais faire un retour sur une conf par évènement, celles qui m’ont le plus intéressé.


Breizhcamp - mise en place d’un ADR

Cela faisait quelque temps que j’avais lu des articles sur comment tracer les décisions d’architecture pour un logiciel. Le sujet m’intéressait fortement car j’y voyais un avantage majeur (ou complémentaire) par rapport aux specs techniques traditionnelles : cela permet de remettre du contexte dans les prises de décision.

Un [ADR](https://www.infoq.com/fr/articles/architecture-decision-records/) (Architecture Decision Record) est un outil qui permet de tracer les choix d’architecture tout au long d’un projet, avec pour chaque décision :

    la date
    les personnes ayant participé au choix
    les différentes options proposées
    les raisons du choix (ou du non choix s’il est décidé de rester en l’état)

Sébastien Lecacheur nous a présenté lors du Breizhcamp 2023 [un retour d’expérience](https://www.youtube.com/watch?v=lvIy9MCw7oI&list=PLv7xGPH0RMUQC6eKGeEXO4PzvKdsU7z2j&index=23) sur la méthode, avec un outil et son usage au sein de sa société. Ils travaillent dans un environnement avec du legacy et pas mal d’applications. Un des problème récurrent est que lorsqu’un nouveau développeur arrive sur un projet, au bout de quelques temps il pose des questions sur l’architecture et certains choix techniques. Si la personne connaissant le mieux l’historique n’est pas là, cela peut générer de la frustration (pourquoi ont-ils fait ça ?). L’un des buts de la mise en place de l’outil était donc de pouvoir apporter facilement des réponses fiables à ce type d’interrogation.

Un autre objectif était de rendre les décisions plus collégiales, de pouvoir intégrer et tracer plus facilement les idées de chacun. Que l’architecture ne soit plus le graal des leads/architectes, mais que chacun puisse avoir son mot à dire. Aujourd’hui, après un certain temps d’utilisation de l’outil et sa diffusion en interne, plus de personnes s’y intéressent et font des propositions. Les équipes s’approprient bien l’outil et l’évolution de l’architecture est devenue plus collaborative.

L’outil qu’il nous a présenté est celui qu’ils ont mis en place : [log4brains](https://github.com/thomvaill/log4brains). C’est un outil NodeJS facile à installer et à déployer, et qui permet de générer un site statique à partir de fichiers markdown. Chaque fichier correspond a une proposition de changement d’architecture, et on peut y tracer toutes les informations nécessaires. Ensuite grâce à la commande suivante on génère le site :

$ log4brains init

La cli vous guide ensuite pour la configuration initiale de votre ADR par rapport au contexte du projet. Une fois cette étape terminée, vous pouvez lancer le site pour voir à quoi cela ressemble :

$ log4brains preview

Voici une capture de l’accueil du site juste après la configuration initiale :
Accueil du site généré par log4brains : une explication du fonctionnement de l’outil

Et voici une autre d’une décision échue :
Premier ADR créé automatiquement : utiliser log4brains pour gérer les ADRs

Comme vous pouvez le constater, l’interface est simple, l’historique des décisions se présente sous forme de frise chronologique à gauche, le modèle de base liste les principales informations, et en bas des liens pour pouvoir naviguer de décision en décision.

Nous l’avons mis en place sur nos projets en cours, nous sommes satisfaits de sa simplicité et il convient à notre usage, après une adaptation du modèle de base de décision (nous utilisons un modèle très simple). Voici [un repo github](https://github.com/joelparkerhenderson/architecture-decision-record) avec plein d’autres informations si cela vous intéresse d’aller plus loin.

Je remets le lien vers la vidéo du talk : [ici](https://github.com/joelparkerhenderson/architecture-decision-record).


Devfest Perros - Copilot : L’intelligence artificielle au service des développeurs

Depuis l’arrivée de GPT3 et ChatGPT, l’IA est devenue un sujet majeur dans la communauté tech, mais également dans la société en général. Github Copilot est disponible pour tous depuis plus d’un an maintenant, cette conférence était donc pour moi l’occasion de découvrir un peu plus le produit et ses capacités.

Tugdual Grall, ingénieur chez Github, nous présente ce talk sur le fonctionnement de Copilot et son usage au quotidien. Il commence par nous présenter un peu ce qui se cache derrière Copilot :

    le moteur est entraîné 2 à 4 fois par an sur tout ce qui est public dans Github
    Il y a un filtre toxique pour modérer le prompt (par exemple “je voudrais faire un outil de DDOS” ne donnera rien)
    l’accent est mis sur la sécurisation et l’optimisation du code suggéré
    il existe un paramètre pour détecter la duplication de code provenant d’un repo public et éviter d’avoir une suggestion trop proche voire identique à ce code
    pour générer une suggestion, un contexte est envoyé à Copilot, rien n’est gardé (session éphémère)
    tous les langages présents sur Github sont supportés

Voici pour les généralités. Passons aux démos. Une fois l’extension installée (dans VsCode ou IntelliJ), Copilot peut nous faire des suggestions :

    de manière automatique, par rapport à la position du chariot dans le fichier ouvert
    via l’écriture d’un commentaire décrivant ce que l’on souhaite faire (en langage naturel, le français fonctionne)
    ou via Copilot Chat

Copilot peut nous assister dans des tâches simples, mais pour la réalisation du code métier il ne sera pas pertinent. Il peut nous aider également via la commande “explain” à comprendre un bout de code, voir nous suggérer des améliorations (sécurité, optimisation). Pour créer son contexte, Copilot utilise les fichiers actuellement ouverts dans l’éditeur. Meilleur est le contexte, meilleures seront les suggestions.

En fin de présentation il nous a également parlé de Copilot for CLI qui permet d’utiliser Copilot dans un terminal. Le fonctionnement reste le même, il permet de générer des suggestions de commande, ou de les expliquer.

Depuis cette présentation j’ai essayé Copilot, au quotidien dans mes activités de développement. Voici quelques retours après plusieurs mois :

    Copilot peut nous faire gagner du temps sur des tâches simples, sur l’écriture de tests ou de la documentation. Avoir le bon contexte, les bons fichiers ouverts dans l’IDE fait la différence.
    Sur le code métier, peu de suggestions sont réellement pertinentes, le métier est surtout du ressort du développeur (et heureusement)
    La commande explain peut être très utile, quand on en a pris l’habitude
    Un point important pour moi, il n’est pas forcement à mettre dans toutes les mains, ou alors avec une formation et une mise en garde. En tant que senior, j’ai vu des suggestions pas du tout adaptées à notre code apparaître, je ne suis pas sûr qu’un débutant aurait forcement vu les soucis. La mise en place de l’outil doit être accompagnée d’une formation et d’un suivi des réalisations (mais tout le monde fait des revues de code 🙂 ). Il ne fait que des suggestions, la maîtrise et la compréhension du code final restent la responsabilité du développeur.

En tant qu’assistant au développement, je suis satisfait de ce qu’il peut m’apporter au quotidien, et pour certaines tâches le gain de productivité est évident.

La conférence n’a pas été filmée, mais vous pouvez trouver l’article de blog de Tugdual sur son usage de Copilot [ici](https://tgrall.github.io/blog/2023/02/21/copilot-how-i-use-it-why-i-love-it), ainsi que la vidéo de ce talk mais à un autre évènement [là](https://www.youtube.com/watch?v=lhkFXOJFWLo).

Si vous avez aimé cet article ou si vous l’avez trouvé utile, n’hésitez pas à 👏 ou à le partager ! 🙂