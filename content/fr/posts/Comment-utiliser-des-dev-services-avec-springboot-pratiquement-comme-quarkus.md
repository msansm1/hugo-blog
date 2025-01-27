---
title: "Comment utiliser des dev services dans SpringBoot, pratiquement comme Quarkus"
description: "Comment utiliser des dev services dans SpringBoot, pratiquement comme Quarkus, en utilisant TestContainers."
theme_version: '2.8.2'
date: 2024-09-27
publishDate: 2024-09-27
cascade:
  featured_image: '/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green.png'
---

Lorsque j'ai commencé un projet sur [Quarkus](https://quarkus.io/), une des principales fonctionnalités du framework était pour moi les dev services. 
Lorsque vous ajoutez une dépendance (extension) comme une base de données à votre projet,
elle vient avec un service qui va automatiquement gérer le démarrage ou l'arrêt d'un conteneur en même temps que votre application pendant le mode de développement.
Vous avez également des paramètres pour le configurer selon ce que vous souhaitez, c'est vraiment cool !

Lorsque j'ai commencé un nouveau projet SpringBoot, j'ai cherché un moyen d'avoir un environnement local d'une façon équivalente aux dev services de Quarkus.
J'ai trouvé que depuis [SpringBoot 3.1](https://spring.io/blog/2023/06/23/improved-testcontainers-support-in-spring-boot-3-1), il est possible d'avoir une classe de test pour démarrer des conteneurs en même temps que l'application,
et ainsi avec seulement une seule classe vous démarrez votre environnement complet. Trop facile !

Pour pouvoir faire cela, nous avons [TestContainers](https://testcontainers.com/) comme dépendance.
Vous pouvez la déclarer de cette manière si vous utilisez Gradle avec le DSL Kotlin :

```kotlin
testImplementation("org.testcontainers:testcontainers:1.19.7")
```

Nous allons utiliser la librairie pour gérer nos conteneurs de dev services.
Nous allons déclarer un bean pour gérer un conteneur pour une base de données PostgreSQL, 
avec un lien sur un port réseau afin de pouvoir y accéder directement grâce à un outil de gestion de BDD pour vérifier nos données :

```java
@TestConfiguration(proxyBeanMethods = false)
public class ContainersConfiguration {

    @Bean
    @ServiceConnection
    public PostgreSQLContainer<?> postgreSQLContainer() {
        PostgreSQLContainer<?> postgreSQLContainer = new PostgreSQLContainer("postgres:15-alpine")
                .withDatabaseName("springbootdevservices")
                .withUsername("dev")
                .withPassword("pass");
        postgreSQLContainer.setPortBindings(List.of("30000:5432"));
        return postgreSQLContainer;
    }
}
```

Comme vous pouvez le voir, nous déclarons un conteneur avec une image de base, où vous pouvez spécifier la version dont votre projet a besoin.
Nous avons également configuré le nom de la base de données ainsi que l'utilisateur et son mot de passe pour y accéder.
L'annotation `@ServiceConnection` est utilisée par Spring pour configurer automatiquement les paramètres de base de données dans Hibernate.

Maintenant, nous allons créer notre classe `TestSpringbootDevservicesApplication` qui va lancer l'application en mode test avec les conteneurs :

```java
public class TestSpringbootDevservicesApplication {

    public static void main(String[] args) {
        SpringApplication.from(SpringbootDevservicesApplication::main)
                         .with(ContainersConfiguration.class)
                         .run(args);
    }

}
```

Si vous exécutez la classe depuis votre IDE, vous allez voir le conteneur démarrer, puis votre application terminera de démarrer et sera disponible.
Avec ceci, nous avons un exemple simple de comment lancer un conteneur, comme une base de données, en même temps que notre application en "mode développement".

Nous allons ajouter [Flyway](https://www.red-gate.com/products/flyway/) pour gérer notre schéma ainsi que des données de test pour notre base de données locale. Vous pouvez ajouter la dépendance :

```kotlin
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-database-postgresql")
```

Si vous n'avez pas d'outils pour gérer vos migrations de base de données, comme Flyway ou [Liquibase](https://www.liquibase.com/), je vous recommande d'y jeter un coup d'oeil et d'ajouter l'un d'entre eux à vos projets.

Nous allons ajouter un fichier de migration dans le dossier `src/main/resources/db/migration` avec le nom `V0.1__init_schema.sql` pour initialiser notre base de données :

```sql
CREATE TABLE shelf
(
    id   UUID NOT NULL,
    name VARCHAR(255),
    CONSTRAINT pk_shelf PRIMARY KEY (id)
);

CREATE TABLE book
(
    id        UUID NOT NULL,
    title     VARCHAR(255),
    author    VARCHAR(255),
    isbn      VARCHAR(255),
    publisher VARCHAR(255),
    shelf_id  UUID,
    CONSTRAINT pk_book PRIMARY KEY (id)
);
```

Puis, nous ajoutons un autre fichier dans `src/test/resources/db/fixture` avec le nom `V0.1.1__init_data.sql` pour insérer nos données de test :

```sql
INSERT INTO shelf (id, name) VALUES ('93118885-1854-4936-9a45-d9c4c10a2c20', 'Thriller');
INSERT INTO shelf (id, name) VALUES ('d61091cb-6b3a-44e2-b66f-aafac7af7d39', 'SF');

INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('71d9323e-83bd-4bc4-876a-a5107002ddaa', 'Red alert', 'Tom Clancy', '1111-22-3333', 'BH', '93118885-1854-4936-9a45-d9c4c10a2c20');
INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('9b3b9043-853b-4c5c-9c2c-a0c4cba69bca', 'Fade Away', 'Harlan Coben', '2222-33-4444', 'West', '93118885-1854-4936-9a45-d9c4c10a2c20');
INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('a85e21a9-5873-442d-8cd9-3bb4b50191be', 'Dune', 'Franck Herbert', '5555-88-9999', 'Spices', 'd61091cb-6b3a-44e2-b66f-aafac7af7d39');
INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('118c39a6-eddb-4abc-bb07-fea200c0276b', 'Book of Dust', 'Philip Pullman', '888-66-7894', 'Rusty', 'd61091cb-6b3a-44e2-b66f-aafac7af7d39');
```

Après cela, un peu de configuration pour Flyway dans le fichier `application.yml` :

```yaml
spring:
  flyway:
    locations: classpath:db/fixture, classpath:db/migration
```

Dorénavant, si vous démarrez votre application en mode dev, vous verrez dans les logs l'exécution de Flyway au démarrage :

```shell
Creating Schema History table "public"."flyway_schema_history" ...
Current version of schema "public": << Empty Schema >>
Migrating schema "public" to version "0.1 - init schema"
Migrating schema "public" to version "0.1.1 - init data"
Successfully applied 2 migrations to schema "public", now at version v0.1.1 (execution time 00:00.007s)
```

Avec ceci, nous avons une base de données locale qui démarre avec notre application, et un jeu de données inséré donc nous pouvons tester directement notre application.

Pour finir, nous allons installer un autre service, et pour pouvoir communiquer correctement avec lui nous devons ajouter un réseau pour nos conteneurs.
Dans notre exemple, nous allons ajouter [Keycloak](https://www.keycloak.org/) comme gestionnaire d'identité de notre application.

Nous allons utiliser un conteneur préconfiguré, simple à paramétrer et que nous utilisons dans nos autres projets. Vous pouvez ajouter la dépendance :

```kotlin
testImplementation("com.github.dasniko:testcontainers-keycloak:3.4.0")
```

Puis, vous déclarez votre conteneur dans la classe `ContainersConfiguration` de cette manière :

```java
@Bean
public KeycloakContainer keycloakContainer() {
    KeycloakContainer keycloakContainer = new KeycloakContainer().withNetwork(network)
                                                                 .withRealmImportFile("realm-export.json");
    keycloakContainer.setPortBindings(List.of("8080:8080"));
    return keycloakContainer;
}
```

Comme vous pouvez le voir, nous avons lié le port 8080 à notre port local 8080 de façon à pouvoir y accéder à http://localhost:8080.
Nous l'avons également paramétré pour utiliser le même réseau que notre base de données et notre application, ainsi chaque service sera capable de communiquer avec les autres.

Nous avons configuré un fichier de royaume (realm file) à importer, et avec ce fichier notre service Keycloak aura un royaume configuré, prêt à être utilisé.
Dans ce fichier de royaume, nous avons la configuration complète du royaume que nous utilisons, avec les rôles, les utilisateurs et toutes les autres données nécessaires.

Avant de le tester, comme Keycloak va écouter sur le port 8080, nous allons configurer notre application sur le port 8081.
Dans le fichier de test `application.yml`, vous pouvez ajouter :

```yaml
server:
  port : 8081
```

Maintenant, nous allons configurer la sécurité de notre application pour vérifier si les requêtes sont bien toutes authentifiées par notre service Keycloak.

Pour faire cela, nous devons ajouter ces dépendances :

```kotlin
implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
implementation("org.springframework.boot:spring-boot-starter-security")

testImplementation("org.springframework.security:spring-security-test")
```

Nous avons ajouté le starter pour SpringBoot Security et celui pour le serveur OAuth2, comme notre service Keycloak sera utilisé en tant que serveur d'authentification OAuth2.

Si vous avez déjà SpringBoot Security, vous connaissez sa configuration relativement verbeuse.
Depuis SpringBoot 3, la configuration doit être découpée en plusieurs méthodes (ou classes), et non plus dans une seule grande méthode comme avant.
Notre configuration ressemble à ceci :

```java
@RequiredArgsConstructor
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {

    private final JwtAuthConverter jwtAuthConverter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.addFilterBefore(new EncodingFilter(), ChannelProcessingFilter.class);
        http.authorizeHttpRequests(authorizeHttpRequestsConfigurer())
            .cors(httpSecurityCorsConfigurer())
            .csrf(csrfConfigurer())
            .oauth2ResourceServer(oAuth2ResourceServerConfigurer())
            .sessionManagement(httpSecuritySessionManagementConfigurer());
        return http.build();
    }

    private static Customizer<AuthorizeHttpRequestsConfigurer<HttpSecurity>.AuthorizationManagerRequestMatcherRegistry> authorizeHttpRequestsConfigurer() {
        return authorize ->
                authorize.requestMatchers(HttpMethod.GET, "/shelves/**").authenticated()
                         .requestMatchers(HttpMethod.GET, "/books/**").authenticated()
                         .anyRequest().permitAll();
    }

    private static Customizer<CsrfConfigurer<HttpSecurity>> csrfConfigurer() {
        return AbstractHttpConfigurer::disable;
    }

    private Customizer<OAuth2ResourceServerConfigurer<HttpSecurity>> oAuth2ResourceServerConfigurer() {
        return oAuth2 -> oAuth2.jwt(jwtConfigurer());
    }

    private Customizer<OAuth2ResourceServerConfigurer<HttpSecurity>.JwtConfigurer> jwtConfigurer() {
        return jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter);
    }

    private static Customizer<SessionManagementConfigurer<HttpSecurity>> httpSecuritySessionManagementConfigurer() {
        return sessionManagement -> sessionManagement.sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }

    private static Customizer<CorsConfigurer<HttpSecurity>> httpSecurityCorsConfigurer() {
        return cors -> {
            UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
            CorsConfiguration config = new CorsConfiguration();
            config.setAllowCredentials(true);
            config.addAllowedOriginPattern(CorsConfiguration.ALL);
            config.addAllowedHeader(CorsConfiguration.ALL);
            config.addAllowedMethod(CorsConfiguration.ALL);
            source.registerCorsConfiguration("/**", config);
            cors.configurationSource(source);
        };
    }

}
```

Nous avons configuré la sécurité de base (cors, csrf) ainsi que OAuth2.
Comme vous pouvez le voir, les 2 chemins `/shelves/**` et `/books/**` sont restreints aux seuls utilisateurs authentifiés.

Nous avons configuré un JwtAuthConverter qui sera utilisé pour extraire le rôle de l'utilisateur du token JWT de Keycloak.
Nous l'avons fait de cette manière :

```java
@Component
public class JwtAuthConverter implements Converter<Jwt, AbstractAuthenticationToken> {

    private final JwtAuthConverterProperties properties;

    public JwtAuthConverter(JwtAuthConverterProperties properties) {
        this.properties = properties;
    }

    @Override
    public AbstractAuthenticationToken convert(@NonNull Jwt jwt) {
        return new JwtAuthenticationToken(jwt, extractRealmRoles(jwt), getPrincipalClaimName(jwt));
    }

    private String getPrincipalClaimName(Jwt jwt) {
        String claimName = JwtClaimNames.SUB;
        if (properties.getPrincipalAttribute() != null) {
            claimName = properties.getPrincipalAttribute();
        }
        return jwt.getClaim(claimName);
    }

    private Collection<? extends GrantedAuthority> extractRealmRoles(Jwt jwt) {
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        return Optional.ofNullable(realmAccess)
                       .filter(access -> access.get("roles") instanceof Collection)
                       .map(access -> (Collection<?>) access.get("roles"))
                       .stream()
                       .flatMap(Collection::stream)
                       .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                       .collect(Collectors.toSet());
    }
}
```

Vous pouvez voir comment nous parcourons les données du token pour obtenir le rôle de l'utilisateur défini dans Keycloak.
La classe JwtAuthConverterProperties est utilisée pour définir les propriétés :

```java
@Validated
@Configuration
@ConfigurationProperties(prefix = "jwt.auth.converter")
public class JwtAuthConverterProperties {

    private String resourceId;
    private String principalAttribute;

    public String getResourceId() {
        return resourceId;
    }

    public void setResourceId(String resourceId) {
        this.resourceId = resourceId;
    }

    public String getPrincipalAttribute() {
        return principalAttribute;
    }

    public void setPrincipalAttribute(String principalAttribute) {
        this.principalAttribute = principalAttribute;
    }
}
```

Après l'ajout de ces fichiers, nous devons configurer notre application pour qu'elle utilise Keycloak pour vérifier les requêtes entrantes.
Nous ajoutons donc la configuration suivante dans le fichier application.yml de test :

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/springbootdevservices
          jwk-set-uri: ${spring.security.oauth2.resourceserver.jwt.issuer-uri}/protocol/openid-connect/certs
```

Maintenant, vous pouvez de nouveau lancer votre classe main de test, et vérifier si un appel à `/shelves` se trouve rejeté sans bearer token.
Vous pouvez récupérer un token en envoyant une requête vers votre server Keycloak :

*Authentication request to the local Keycloak server with Bruno tool showing the JSON request result with tokens from Keycloak.*
Requête d'authentification vers le serveur local Keycloak avec l'outil Bruno

Puis relancer la requête vers votre endpoint, mais avec le token dorénavant :

*Request to our local service launched with Bruno tool, showing the result of the call with a success response.*
La requête envoyée avec le token, et une réponse HTTP 200 avec des données !

Vous pouvez maintenant exécuter depuis votre IDE des conteneurs en tant que dev services selon les besoins de votre projet, tout en Java avec SpringBoot !

Merci pour la lecture de cet article, j'espère qu'il vous sera utile. Vous pouvez trouver tout le code d'exemple dans ce [repo github](https://github.com/Slickteam/springboot-devservice).
