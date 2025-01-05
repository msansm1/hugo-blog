---
title: "KTOR et Keycloak - authentification avec OpenId"
description: "Comment configurer KTOR pour utiliser Keycloak en tant que serveur OpenID."
# 1. To ensure Netlify triggers a build on our exampleSite instance, we need to change a file in the exampleSite directory.
theme_version: '2.8.2'
date: 2019-11-07
publishDate: 2019-11-07
cascade:
featured_image: '/images/ktor_keycloak_openid/ktor_keycloak_openid_keycloak_login.png'
---

Depuis quelque temps je voulais essayer Kotlin. Lorsque j'ai vu que Jetbrains annonçait le framework KTOR pour créer des applications web,
j'ai vu une bonne opportunité d'en apprendre plus sur le langage tout en découvrant ce nouveau framework.

KTOR est un framework web pour construire des services ou applications entièrement en Kotlin, car il peu également générer du HTML ou du CSS via des DSLs (Domain-Specific Language) en Kotlin.
A la fin de cet article, il y a un exemple de page HTML écrite à l'aide de ce DSL.
Ce framework embarque un serveur web (on peut choisir entre jetty, netty, tomcat ou autre), et il est possible de le configurer dans le code directement.
De plus, comme il est édité par Jetbrains, créateur de Kotlin, il me semblait d'autant plus intéressant.

Pour tester KTOR, j'ai décidé de faire un site intranet sur lequel nous devrons nous connecter avant d'y accéder.
Nous utilisons Keycloak en tant que gestionnaire d'identité, j'ai donc cherché comment l'utiliser avec KTOR.

Comme autre référence, vous pouvez lire la documentation KTOR ou ces deux articles (1 et 2) à propos de OAuth2 et KTOR.
Pour le faire fonctionner correctement avec Keycloak j'ai essayé beaucoup de choses, et j'ai décidé d'écrire cet article pour aider d'autres devs.

Tout d'abord, vous devez avoir un serveur Keycloak et configuré correctement. Je le lance via Docker, mais vous pouvez le lancer avec la version standalone si vous préférez.

```shell
$ docker pull jboss/keycloak
$ docker run -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=password -p 10080:8080 jboss/keycloak
```

Ainsi vous avez un serveur Keycloak qui tourne en local sur le port 10080, avec un compte administrateur créé.
Maintenant vous pouvez vous connecter à l'adresse http://localhost:10080 avec ce compte.

Nous allons d'abord créer un royaume (Realm) puis le configurer selon nos besoins.
Le royaume Master sert à configurer Keycloak lui-même, pas les autres domaines.
Vous pouvez cliquer sur "Master" dans le menu de gauche, puis "Add realm". Nommez-le Ktor.

Maintenant sur cette page :
![Paramètres généraux du royaume Keycloak KTOR](/images/ktor_keycloak_openid/ktor_keycloak_openid_realm_settings.png "Paramètres généraux du royaume Keycloak KTOR")

Créer un nouveau client de cette manière :
![Configuration du client dans Keycloak](/images/ktor_keycloak_openid/ktor_keycloak_openid_client_settings.png "Configuration du client dans Keycloak")

Ici les paramètres importants sont le protocole client et l'URL valide de redirection.
C'est vers cette URL que Keycloak redirigera après le résultat de l'authentification (réussie ou échouée).

Maintenant nous allons créer un role et un utilisateur pour pouvoir se connecter à notre application.
Vous pouvez trouver la méthode ici (cet article m'a servi de base pour ma configuration).

Votre serveur Keycloak est maintenant correctement configuré, place au code !

Dans KTOR, le protocole OAuth2 est intégré au framework, donc la clé réside dans une configuration correcte et une maîtrise des échanges lors d'un appel OAuth2.

Nous allons d'abord définir un fournisseur OAuth2 qui correspond à la configuration de notre serveur Keycloak :
```kotlin
val keycloakAddress = "http://localhost:10080"

val keycloakProvider = OAuthServerSettings.OAuth2ServerSettings(
    name = "keycloak",
    authorizeUrl = "$keycloakAddress/auth/realms/ktor/protocol/openid-connect/auth",
    accessTokenUrl = "$keycloakAddress/auth/realms/ktor/protocol/openid-connect/token",
    clientId = "ktorClient",
    clientSecret = "123456",
    accessTokenRequiresBasicAuth = false,
    requestMethod = HttpMethod.Post, // must POST to token endpoint
    defaultScopes = listOf("roles")
)
val keycloakOAuth = "keycloakOAuth"
```
OAuth2 provider definition

Ici nous avons tous les paramètres nécessaires définis pour connecter notre serveur Keycloak.
L'identifiant client (clientId) et son secret (clientSecret) sont obligatoires pour la configuration dans Ktor (et ils ne peuvent être vides), mais pour Keycloak ils ne sont pas obligatoires vu que nous avons paramétré le type d'accès comme "public" pour le client.
Si vous le configurez comme confidentiel, vous aurez un nouvel onglet dans Keycloak pour obtenir le nom et le secret du client.
Le format des URLs est le même pour tous les serveurs Keycloak, seul l'identifiant client dépend de votre propre configuration.
La dernière valeur "keycloakOAuth" va servir en tant que paramètre dans les appels suivants.

Maintenant nous allons demander à KTOR d'installer la fonctionnalité "Authentication" et la configurer pour notre Keycloak :
```kotlin
install(Authentication) {
    oauth(keycloakOAuth) {
        client = HttpClient(Apache)
        providerLookup = { keycloakProvider }
        urlProvider = {
            redirectUrl("/")
        }
    }
}
```
Installation of authentication feature

Ainsi, vous avez défini une méthode d'authentification à travers votre serveur Keycloak.

Ensuite, nous allons dire à Keycloak que la route vers notre site est protégée par authentification :
```kotlin
authenticate(keycloakOAuth) {
    location<index> {
        param("error") {
            handle {
                call.loginFailedPage(call.parameters
                                         .getAll("error").orEmpty())
            }
        }

        handle {
            val principal = call.authentication
                           .principal<OAuthAccessTokenResponse.OAuth2>()
            if (principal != null) {
                call.loggedInSuccessResponse(principal, call)
            } else {
                call.respond(HttpStatusCode.Unauthorized)
            }
        }
    }
}
```
Protection of location by authentication

Dans ceci nous commençons par déclarer que notre localisation <index> est protégée par authentification (celle qui utilise notre Keycloak).
Cette partie va à l'intérieur du bloc de déclaration des routes dans la fonction principale de KTOR.

Après nous déclarons notre chemin vers l'index :
```kotlin
 @Location("/")
 class index
```
Location declaration

Puis nous pouvons avoir 2 fonctions pour gérer si l'authentification est réussie ou a échouée :
```kotlin
private suspend fun ApplicationCall.loginFailedPage(errors: List<String>) {
    respondHtml {
        head {
            title { +"Login with" }
        }
        body {
            h1 {
                +"Login error"
            }
            for (e in errors) {
                p {
                    +e
                }
            }
        }
    }
}

private suspend fun ApplicationCall.loggedInSuccessResponse(
    callback: OAuthAccessTokenResponse.OAuth2,
    call: ApplicationCall
) {
    val jwtToken = callback.accessToken
    val token = JWT.decode(jwtToken)
    val name = token.getClaim("name").asString()
  
    respondHtml {
        head {
            title { +"Login with" }
        }
        body {
            h1 {
                +"Login successfull !"
            }
            h2 {
                +"Welcome "
                +"$name"
                +" !!"
            }
        }
    }
}
```
Functions for failed / successful login

Dans cette fonction loginInSuccessResponse j'ai extrait le nom de l'utilisateur du jeton JWT pour l'afficher dans ma page HTML.

Ensuite vous pouvez démarrer notre application KTOR, et lorsque vous essayez d'accéder à votre page d'index "/" vous êtes redirigés vers votre serveur Keycloak pour vous authentifier.
![Portail d'authentification de Keycloak](/images/ktor_keycloak_openid/ktor_keycloak_openid_keycloak_login.png "Portail d'authentification de Keycloak")
Keycloak authentication portal

Et après vous arrivez bien dans la partie privée de votre application !
![Maintenant vous êtes authentifiés !](/images/ktor_keycloak_openid/ktor_keycloak_openid_login_successfull.png "Maintenant vous êtes authentifiés !")

En conclusion, je pense que le framework est assez intéressant, malgré sa jeunesse.
La communauté est relativement petite actuellement, mais j'espère qu'elle grandira aec le temps.
Vous pouvez configurer votre serveur directement dans le code et ajouter des fonctionnalités est relativement simple.
Avec Kotlin, si quelque chose vous manque vous pouvez l'ajouter assez facilement à l'aide des fonctions d'extension (extension functions).
Alors que Jetbrains continue de l'améliorer (à ce jour il est en version 1.3.0 bêta) KTOR peut être une bonne option si vous cherchez un framework web en Kotlin.
