---
layout: post
title: Spring Security OAuth2 Second Generation
author: Haytham Mohamed
---

Spring framework provides a comprehensive and extensible authentication
and authorization support. Latest enhancements in Spring 5.x have made it simple
to apply security standards such as OAuth2 to secure applications. In this blog I will
demonstrate how to use OAuth2 second-generation support in Spring framework to
secure a distributed and reactive-based microservices application.

## Architecture
----

A "Flight Agency" demo application will be used in this
blog to illustrate applying the latest Springframework 5.x security features of OAuth2 to
help securing a distributed microservices application. The architecture of the
demo application consists of a front-end web application and a couple
of backend microservices. All modules of the application are implemented using
Spring Boot to take advantage of the auto-configuration and opinionated features.
All applications are also implemented with reactive WebFlux support provided by
Springframework.

As illustrated in the diagram below, the system consists of:

* Front-end Web application implemented using Tymeleaf. The application allows
a user to search for a flight from an origin to a destination within
certain dates, select and book the itinerary flights (called agency-web).

* A back-end service to retrieve available itinerary flights (called flights-service).
* A back-end service to perform reservations and book the itinerary (called reservations-service)

Additionally, the application demo uses common services implemented with Spring
cloud (Spring Boot based), such as:

* A registration and discovery service using Spring Cloud Eureka implementation to
help service to register and discover each other.
* A Spring Cloud Gateway, as an edge proxy in front of the two backend services
of flights and reservations.

<img src="../images/spring-oauth2-gen2/RA-OAuth2-2.png" width="50%" height="50%" title="architecture">

## Running the Application
----

Follow the steps below to run the application:

* clone the application from github

``` shell
$ git clone https://github.com/Haybu/RA-OAuth2-Gen2.git
```

* Change your working directory to go to "RA-OAuth2-Gen2" and run the startup.sh
script file to compile, package and run all application's modules

``` shell
$ ./startup.sh
```

## Application Flow
----

After bringing all modules up, access the web client application by going to
http://localhost:8080. You will be directed to the OAuth2 authorizatio server (UAA) to authenticate and authorize to the application. Once successfully
logged in, search for a trip intinerary by entering: origin as __AUS__, destination
as __IAH__, departure date as __5/5/2018__ and return date as __5/22/2018__.

The application will display a list of flights, select your outbound flight. On
the next page pick your inbound flight, then click on "book" on the following
page and you will be presented with two confirmation numbers for each flight.

As illustrated in the sequence diagram below, the client application
utilizes a backend "flights service" to search for available flights per a user search request. Additionally, a "reservations" backend service is also helping a
client to book the selected flights. The client application
and both backend services are implemented using Spring Boot. They all leverage
Spring Cloud Eureka for services registrations and discoveries. The front-end client
application talks to the backend services via a Spring Cloud Gateway.

<img src="../images/spring-oauth2-gen2/RA-OAuth2-seq-diagram-1.png" width="460" height="400" title="architecture">

## Services Registration and discovery
----

Client application, two backing API services and the gateway application  
register themselves automatically with a Spring Cloud Eureka server when they
spin up. When the client application wants to interact with one of the backend
services it uses a "WebClient" bean that is set with a "LoadBalancerExchangeFilterFunction"
filter to help with balancing out of the discovered target services' instances.

For example, a client application defines a load-balanced "WebClient" bean
as listed below and uses that to interact with underneath services discovered
from a Eureka service. Further configuration to the "WebCliet" bean will be
defined further below to help authenticating and access the registered backend
services.

~~~ java
@Bean
 WebClient webClient(LoadBalancerExchangeFilterFunction eff) {
		ServerOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
		      new ServerOAuth2AuthorizedClientExchangeFilterFunction(repo1, repo2);

		return WebClient.builder()
				.filter(eff)
				.build();
 }
~~~

## Spring Cloud Gateway
----

As mentioned above, the web client application communicates with the two backend
micorservices via a Spring Cloud Gateway. The Gateway is configured also with all the routes qualified by a target path to each proxied service. And As shown below, a "RewritePath" filter
is defined to observe the context path of each target service.

~~~ yaml
spring:
  cloud:
    gateway:
      routes:
      - id: flights_service_route
        uri: lb://flights-service
        predicates:
        - Path=/api/flights/**
        filters:
        - RewritePath= /api/flights/(?<segment>.*),/flights/$\{segment}
      - id: reservations_service_route
        uri: lb://reservations-service
        predicates:
        - Path=/api/reservations/**
        filters:
        - RewritePath= /api/reservations/(?<segment>.*),/reservations/$\{segment}
~~~

## Security
----

In this architecture we need to apply Spring security to the client application
and the two backend microservices. New security enhancement in spring 5.x paves
an easy way to secure applications using OAuth2 standards.

### 1. Web Application - OAuth2 Client

You can take advantage of auto-configuring clients with Spring Security if you have
"spring-security-oauth2-client" dependency in your classpath. It is even getting
simpler to register as many clients as you desire using properties with
"spring.security.oauth2.client" prefix, and define your provider's properties with
"spring.security.oauth2.provider" prefix.

For instance, in this demo, after the web client application is registered with
one of the OAuth2 / OpenID Connect providers (like UAA), you can configure the
web application with properties that register a client with the a client-id and
client-secret. Also you can define the selected client's provider with information
such as the authorization URI, token URI, user-info URI and JWK set URI. With those
providers who support OpenID Connect discovery, the configuration could be further
simplified by specifying the issuer URI (with "spring.security.oauth2.client.provider.
oidc-provider.issuer-uri" prefix).

~~~ yaml
spring:
  security:
    oauth2:
      client:
        registration:
          login-client:
            client-id: login-client
            client-secret: secret
            client-authentication-method: basic
            scope: openid,profile,email
            authorization-grant-type: authorization_code
            client-name: Login Client
            redirect-uri-template: "{baseUrl}/login/oauth2/code/{registrationId}"
            provider: uaa            
        provider:
          uaa:
            authorization-uri: http://localhost:8099/uaa/oauth/authorize
            token-uri: http://localhost:8099/uaa/oauth/token
            user-info-uri: http://localhost:8099/uaa/userinfo
            user-name-attribute: sub
            jwk-set-uri: http://localhost:8099/uaa/token_keys
~~~

The configuration listed above shows many properties for clarity.

### 2. Reservations Service and Flights Service as Resource Servers

You can designate and configure a resource server in Spring Security 5.x by having "spring-security-oauth2-resource-server" in your classpath. Spring Boot can set up
an OAuth2 Resource Server as long as JWK Set URI or OIDC Issuer URI is specified.

For example, the flight service acts as a resource server and is
configured using the properties below. With this set up the client application
can communicate securely with the backend flights service.

~~~ yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:8099/uaa/token_keys
~~~

### 4. Spring Cloud Gateway - Resource Server

The Spring Cloud Gateway in this architecture acts also as an edge resource
service to the frontend web application. It is set with
resource server configuration properties as the other two backend services.
With no further configuration, Spring Cloud Gateway passes through the Authentication
token it receives from the client application as an "Authorization" Bearer header.

## Conclusion
----

The Spring security team at Pivotal made an awesome job to help
securing clients and resource servers easily with Spring Security 5.x. One can easily secure applications using OAuth2 standards with so minimum fuss.
You can get it all configured by including the appropriate dependencies and
right OAuth2 configuration properties for the registered client, resource server
or optionally a provider.

## Resources
----

* Sample Demo application Code in [Github](https://github.com/Haybu/RA-OAuth2-Gen2)
* Spring Security [Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-security.html)
