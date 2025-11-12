---
title: "KTOR and Keycloak - authentication with OpenId"
description: "How to configure KTOR to use Keycloak as OpenID server."
date: 2019-11-07
publishDate: 2019-11-07
featuredImage: '/images/ktor_keycloak_openid/ktor_keycloak_openid_keycloak_login.png'
code:
  copy: true
---

For some time I wanted to try Kotlin, and when I saw that Jetbrains announced the framework KTOR for web applications, it seems to me like a good opportunity to learn more about the language and to discover this new framework.

KTOR is a web framework for building web services or applications entirely in Kotlin, since you can generate HTML or CSS files in Kotlin with DSLs

(Domain-Specific Language). At the end oh this article there is an example of an HTML page written with the Kotlin HTML DSL. It embeds a web server (which you can choose between jetty, netty, tomcat or others), and you can configure it directly in your application’s code. Since it is edited by Jetbrains (creator of the Kotlin language), it seemed pretty interesting to me.

To try and test this framework, I decided to make an internal website where we must authenticate ourselves before acceding the content. Our identity management is done by Keycloak, so I searched how to make it work with KTOR.

As another good reference, you can read the KTOR documentation or these articles (1 & 2) about OAuth2 and KTOR. To make it work correctly with Keycloak I have tried many things, and I decided to write this article so it can help others developers.

First of all, you must have a Keycloak server and configure it correctly. I will run it on docker, but you can download the standalone release if you prefer.

```shell
$ docker pull jboss/keycloak
$ docker run -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=password -p 10080:8080 jboss/keycloak
```

Like this you have a Keycloak server running locally on the 10080 port, with an administrator account created. Now login at http://localhost:10080 with your admin account.

We will create first a new Realm in order to configure it as we want. The master Realm is for configuring Keycloak itself, not other domains. You click on “Master” on the left menu, then “Add realm”. Name it Ktor.

Now on this page:
![General settings of Keycloak Ktor Realm](/images/ktor_keycloak_openid/ktor_keycloak_openid_realm_settings.png "General settings of Keycloak Ktor Realm")


Create a new client like this:
![Configuration for client on Keycloak](/images/ktor_keycloak_openid/ktor_keycloak_openid_client_settings.png "Configuration for client on Keycloak")


Here the important parameters are the client protocol and the Valid Redirect URL. It’s that URL that Keycloak will redirect to after login result (successful or failed).

Now we will create a role and a user in order to allow to login into our application. You will find the method here (I based my configuration on this article).

Your Keycloak server is now correctly configured, let’s code !

In KTOR, the OAuth2 protocol is integrated to the framework, so the keys lie in a good configuration and mastering the flow of an OAuth2 call.

We will first define an OAuth2 provider according to our Keycloak server configuration :
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

Here you have all the necessary parameters defined to connect to your Keycloak server. The clientId and clienSecret are mandatory for the configuration in KTOR (and must not be empty), but for Keycloak they are not mandatory since we set an access type for the client as “public”. If you set it to “confidential”, you will have a new tab in Keycloak to get the name and the secret of the client. The form of the URLs are the same for all Keycloak servers, the clientId depends of your own configuration. The last value keycloakOAuth will serve as parameter in later calls.

Now we will ask KTOR to install the Authentication feature and to configure it for our Keycloak:
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

Like that, you have defined an authentication method through your Keycloak server.

We will now tell KTOR that our route to our site is protected by authentication:
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

In this we begin by declaring that our location<index> is protected by authentication (that is using our Keycloak). This part goes inside the routing declaration of the main function for KTOR.

After that we declare our index path:
```kotlin
 @Location("/")
 class index
```
Location declaration

Then we can have the two functions to manage if the login is successful/failed:
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

In the loginInSuccessResponse function I have extracted the user’s name from the JWT token to display it in the HTML page.

After that you can start your KTOR application, and when you try to access your index page “/” you will be redirected to your keycloak server to authenticate yourself.
![Keycloak authentication portal](/images/ktor_keycloak_openid/ktor_keycloak_openid_keycloak_login.png "Keycloak authentication portal")


And after that you are in the private part of your application !
![Now your are authenticated !](/images/ktor_keycloak_openid/ktor_keycloak_openid_login_successfull.png "Now your are authenticated !")


In conclusion, I think the KTOR framework is quite good, despite being young. The community is quite small today, but I hope it will grow as time will pass. You can configure your Web server directly in the code, adding functionalities is quite simple. With Kotlin, if something is missing in the framework functions you can add it easily with extension functions. While Jetbrains continues to improve it (at this day there is a beta version of 1.3.0 release), this framework can be a good option if you are looking for a web framework in Kotlin.
