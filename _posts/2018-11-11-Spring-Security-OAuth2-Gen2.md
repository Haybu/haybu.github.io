---
layout: post
title: Spring Security OAuth2 Second Generation
author: Haytham Mohamed
---

Spring framework provides a comprehensive and extensible support for authentication
and authorization. Latest enhancements in Spring 5.x have made it simple
to apply security standards such as OAuth2 to secure applications. In this blog I will
demonstrate how to use OAuth2 second-generation support in Spring framework to
secure a distributed microservices application.

## Architecture
----

A simple "Flight Agency" demo application will be used in this
blog to illustrate using the latest Spring 5.x security features of OAuth2 to
support microservices authentication and authorization. The architecture of the
distributed demo application consists of a front-end web application and a couple
of backend microservices. All modules of the application are implemented using
Spring Boot to take advantage of the auto-configuration and opinionated features.

As illustrated in the diagram below, the system consists of:

* Front-end Web application implemented using Tymeleaf. The application allows
a user to search for a flight from an origin to a destination within
certain dates, select and book the itinerary flights (called agency-web).

* A back-end service to retrieve available itinerary flights (called flights-service).

* A back-end service to perform reservations and book the itinerary (called reservations-service)

Additionally, the application demo uses common services implemented with Spring
cloud (Spring Boot based), such as:

* A configuration server to externalize and centralize microservices' configurations.

* A registration and discovery service using Spring Cloud Eureka implementation to
help service to register and discover each other.

* A Spring Cloud Gateway, as an edge proxy in front of the two backend services
of flights and reservations.

<img src="../images/spring-oauth2-gen2/RA-OAuth2-2.png" width="50%" height="50%" title="architecture">

## Running the application
----

Before you run the application you need to register a client in one of the OAuth2
/ OpenId Connect providers. Okta can be chosen as a provider for this demo. You can
create a free developer account if you do not have one already by going to okta
developer's [portal] (https://developer.okta.com/). Once logged into okta, register
a new application with authorization_code grant, identify a name and specify the following
parameters in the registration form:

* Login redirect URIs: http://localhost:8080/login/oauth2/code/okta
* Initiate Login URI: http://localhost:8080

Note down the client ID and client secret provided after you complete the registration.

Register another application but this time with client_credentials grant, specify
a name in the registration form and note the provided client ID and client secret.

* clone the application from github

``` shell
$ git clone https://github.com/Haybu/RA-OAuth2-Gen2.git
```

* Change your working directory to go to "RA-OAuth2-Gen2/config-files" folder, edit
agency-web.yml file and enter the client_id and client secret of the application
registered in okta with authorization_client grant type. Also enter your okta's
account URL on "spring.security.oauth2.provider.okta.*" properties to replace the
host name portions as appropriate.

* On the same folder, edit reservations-service.yml file and enter the client ID
and client secret of the application registered in okta with client_credentials
grant type. Also make sure to supply appropriate URL per your Okta's account on
the security provider's parameters.

* change your working directory back to "RA-OAuth2-Gen2" and run the startup.sh
script file to compile, package and run all application's modules

``` shell
$ ./startup.sh
```

## Application Flow
----

After bringing all modules up, access the web client application by going to
http://localhost:8080. You will be directed to your OAuth2 provider's (in this
case Okta) to authenticate and authorize to the application. Once successfully
logged in, search for a trip intinerary by entering: origin as __AUS__, destination
as __IAH__, departure date as __5/5/2018__ and return date as __5/22/2018__.

The application will display a list of flights, select your outgoing flight. On
the next page pick your returning flight, then click on "book" on the following
page and you will be presented with two confirmation numbers for each flight.

As illustrated in the sequence diagram below, the client application
utilizes a backend "flights service" to provides all available flights per
a search request. Additionally, a "reservations service" is also helping the
client application to perform booking on the selected flights. The client application
and both backend services are implemented using Spring Boot. They all leverage
Spring Cloud configuration server to externalize their configurations, and Spring
Cloud Eureka for services registrations and discoveries. The front-end client
application talks to the backend services via a Spring Cloud Gateway.

<img src="../images/spring-oauth2-gen2/RA-OAuth2-seq-diagram-1.png" width="460" height="400" title="architecture">

## Spring Configuration Server
----

While you can choose from many supported backend to host application configurations,
in this demo a local native file system storage is used to keep all application's
"yml" property files. The configuration files repository, illustrated below, contains
all "yml" files each named after its application name.

<img src="../images/spring-oauth2-gen2/config-files.png" width="200" height="120" title="architecture">

Applications bootstrap their configurations by pointing to the configuration
server URL as identified in each application's ./resources/bootstrap.yml file.

## Services Registration and discovery
----

Client application, two backing API services and the gateway application  
register themselves automatically with a Spring Cloud Eureka server when they
spin up. When the client application wants to interact with one of the backend
services it uses a "WebClient" bean that is set with a "LoadBalancerExchangeFilterFunction"
filter to help with balancing out of the discovered target services' instances.

For example, a client application would define a load-balanced "WebClient" bean
as listed below and use that to interact with underneath services discovered
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

## Web Application - OAuth2 Client
----
