---
title: "How to use dev services in SpringBoot, almost like Quarkus"
description: "How to configure and use dev services in SpringBoot, almost like Quarkus, using TestContainers."
date: 2024-09-27
publishDate: 2024-09-27
featuredImage: '/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green.png'
code:
  copy: true
---

When I started a projet on [Quarkus](https://quarkus.io/), one of the big feature for me with this framework was the dev services. When you add a dependency (extension) like a database on your projet, it comes with a service that will manage automatically the start/stop of a container along with your application in development mode. You also have configuration parameters to set it the way you want, it is truly great !

When I started a new project on SpringBoot, I searched a way to have my local environment running in a way like the dev services of Quarkus. I found that since [SpringBoot 3.1](https://spring.io/blog/2023/06/23/improved-testcontainers-support-in-spring-boot-3-1), you can have a test class to start containers along with your application, and thus with only one class start your complete environment. So easy !

To be able to do that, we must have [TestContainers](https://testcontainers.com/) as dependency. It will manage the container for us. You can declare it like this if you are using Gradle with Kotlin DSL:

testImplementation("org.testcontainers:testcontainers:1.19.7")

We can use the library to manage our dev services containers. We will declare a bean to manage a container for a PostgreSQL database, with a network port bind so we can access directly with a database tool to check our data:

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

As you can see, we declare our container with a base image, where you can set the version needed by your project. We set also the name of the database and the user / password to access it. The annotation @ServiceConnection is used by Spring to configure automatically the database parameters in Hibernate.

Now, we will create our new TestSpringbootDevservicesApplication class that will launch the application in dev mode with the containers:

public class TestSpringbootDevservicesApplication {

public static void main(String[] args) {
SpringApplication.from(SpringbootDevservicesApplication::main)
.with(ContainersConfiguration.class)
.run(args);
}

}

If you run this class in your IDE, you will see the container start, then your application finish to start and is available. With this we have a simple example how to run a simple container, like a database, along our application in development mode.

We will now add [Flyway](https://www.red-gate.com/products/flyway/) to manage our schema and fixtures in our local database. You can add the dependency:

implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-database-postgresql")

If you don’t have a tool to manage your database migrations, like Flyway or [Liquibase](https://www.liquibase.com/), I recommend you to take a look at them and to add one of them to your projects.

We will add a migration file in the src/main/resources/db/migration with the name V0.1__init_schema.sql to initiate the schema of our database:

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

Then, we will add a file in src/test/resources/db/fixture with the name V0.1.1__init_data.sql to insert fixtures data in our database:

INSERT INTO shelf (id, name) VALUES ('93118885-1854-4936-9a45-d9c4c10a2c20', 'Thriller');
INSERT INTO shelf (id, name) VALUES ('d61091cb-6b3a-44e2-b66f-aafac7af7d39', 'SF');

INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('71d9323e-83bd-4bc4-876a-a5107002ddaa', 'Red alert', 'Tom Clancy', '1111-22-3333', 'BH', '93118885-1854-4936-9a45-d9c4c10a2c20');
INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('9b3b9043-853b-4c5c-9c2c-a0c4cba69bca', 'Fade Away', 'Harlan Coben', '2222-33-4444', 'West', '93118885-1854-4936-9a45-d9c4c10a2c20');
INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('a85e21a9-5873-442d-8cd9-3bb4b50191be', 'Dune', 'Franck Herbert', '5555-88-9999', 'Spices', 'd61091cb-6b3a-44e2-b66f-aafac7af7d39');
INSERT INTO book (id, title, author, isbn, publisher, shelf_id) VALUES ('118c39a6-eddb-4abc-bb07-fea200c0276b', 'Book of Dust', 'Philip Pullman', '888-66-7894', 'Rusty', 'd61091cb-6b3a-44e2-b66f-aafac7af7d39');

After that, a little configuration for Flyway in the application.yml file:

spring:
flyway:
locations: classpath:db/fixture, classpath:db/migration

Now, if you start your application in dev mode, you will see in the logs the file run by Flyway at start:

Creating Schema History table "public"."flyway_schema_history" ...
Current version of schema "public": << Empty Schema >>
Migrating schema "public" to version "0.1 - init schema"
Migrating schema "public" to version "0.1.1 - init data"
Successfully applied 2 migrations to schema "public", now at version v0.1.1 (execution time 00:00.007s)

With this, we have a local database starting with our application, and data inserted so we can directly test our application.

At last we will add a new service, and to be able to communicate correctly with it we must add a network for our containers. For this example we will add [Keycloak](https://www.keycloak.org/) as identity manager for our application.

We will use a preconfigured container, it is simple to configure and we use it on our projects. You can add this dependency :

testImplementation("com.github.dasniko:testcontainers-keycloak:3.4.0")

Then we will declare our container like this in the ContainersConfiguration class:

@Bean
public KeycloakContainer keycloakContainer() {
KeycloakContainer keycloakContainer = new KeycloakContainer()
.withNetwork(network)
.withRealmImportFile("realm-export.json");
keycloakContainer.setPortBindings(List.of("8080:8080"));
return keycloakContainer;
}

As you can see, we bind the port 8080 to our local 8080 port so we can access it on http://localhost:8080. We also set it to use the same network as our database and our service, so each service will be able to communicate with the others.

We set a realm file to import, and with that file our Keycloak service will have a configured realm to use. In this realm file we have the complete configuration for the realm we use, with roles, users, and any other data you need.

Before testing it, since Keycloak is listening on 8080, we will set our application on 8081. In the application.yml file for the tests, you can add:

server:
port : 8081

Now we will configure the security on our application to check if all the requests are authenticated on our Keycloak service.

To be able to check authentication using our Keycloak service, we need to add these new dependencies:

implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
implementation("org.springframework.boot:spring-boot-starter-security")

testImplementation("org.springframework.security:spring-security-test")

We added the main starter for SpringBoot security and the specific starter for OAuth2 server, since our Keycloak service will be used as an OAuth2 authentication server.

If you already know Spring Security, you know that its configuration is quite verbose. Since SpringBoot 3, the configuration must be done in multiple methods (or classes), not in one big single method like before. Our configuration looks like this:

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

We have configured basic security (cors, csrf) and the configuration for OAuth2. As you can see, the 2 paths /shelves/** and /books/** are restricted to only authenticated users.

We have set a JwtAuthConverter that will be use to extract the role from the JWT token of Keycloak. This is the way we have done it:

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

You can see how we search through the data of the token to get the role of the user defined in Keycloak. The JwtAuthConverterProperties is simply used to define properties:

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

After adding these files, we must configure our application so it uses our Keycloak server to check input requests. So we add this configuration in the application.yml file for tests :

spring:
security:
oauth2:
resourceserver:
jwt:
issuer-uri: http://localhost:8080/realms/springbootdevservices
jwk-set-uri: ${spring.security.oauth2.resourceserver.jwt.issuer-uri}/protocol/openid-connect/certs

Now you can run again the main test class, and check if the call to /shelves gets rejected without bearer token. You can get a token by requesting it on the Keycloak server:
Authentication request to the local Keycloak server with Bruno tool showing the JSON request result with tokens from Keycloak.
Authentication request to the local Keycloak server with Bruno tool

Then request again the endpoint, but with the token now :
Request to our local service launched with Bruno tool, showing the result of the call with a success response.
The request send with the token, and an HTTP 200 response with data !

Now you can run locally within your IDE containers as dev services for your project needs, all in Java with SpringBoot !

Thank you for reading all this article, I hope it can be useful to you. You can find all the code in this [github repo](https://github.com/Slickteam/springboot-devservice).

If you liked this article or you found it useful, don’t hesitate to 👏 it or to share it ! 🙂