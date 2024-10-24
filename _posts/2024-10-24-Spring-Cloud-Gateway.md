---
layout: single
title: Spring Cloud Gateway
excerpt: "Si gestionas microservicios, seguramente te has encontrado con la necesidad de manejar autenticación, balanceo de carga y control de peticiones de forma eficiente. En este tutorial, te mostraré cómo Spring Cloud Gateway puede ayudarte a simplificar y optimizar estas tareas clave en tu arquitectura de microservicios, sin complicaciones."

date: 2024-10-24
classes: wide
header:
  teaser: /assets/images/Spring_Cloud_Gateway/springCloudIcon.webp
  teaser_home_page: true
  icon: /assets/images/Spring_Cloud_Gateway/springCloudIcon.webp
categories:
  - Spring Cloud
  - Spring Cloud Gateway
  - Gateway
  - SpringBoot
tags:  
  - SpringBoot
---

## Porqué tener un API Gateway
Imagínate que tienes una aplicación montada con microservicios, cada uno de estos microservicios está expuesto en un puerto para poder recibir requests.

El microservicio de events devolverá la información relacionada con los eventos, y el microservicio de bookings devolverá información sobre las reservas ... 
![](/assets/images/Spring_Cloud_Gateway/Pasted image 20241017183314.png)
Por ahora ningún problema. Pero supongamos que ahora queremos `añadir autenticación`, uff , hay que añadirlo en cada uno de los microservicios. O supongamos que queremos `controlar el número de peticiones` que recibimos en nuestra aplicación, se complica verdad? Pues para poner peor la situación, en un entorno real no habrán 2 microservicios sino decenas

Por estas y por muchas más razones (que no comento ya que si no el blog se extendería en exceso, y ya nadie lo leería😢 ) necesitamos un `Gateway`, este será el portero por el que cada una de las peticiones deberá pasar antes de poder llegar a un microservicio.

![](/assets/images/Spring_Cloud_Gateway/Pasted image 20241017183848.png)
Y de verdad te digo que es muy sencillo montar este Gateway, sobre todo gracias a `Spring Cloud Gateway`


## Spring Cloud Gateway
Spring Cloud Gateway nos hace la vida tremendamente fácil, y si ya sabes implementar `Eureka` como vimos en el otro [blog](https://egarmar1.github.io/Eureka-y-openFeign-en-Microservicios/), entonces está chupado.

Antes de crear tu API Gateway déjame comentarte una funcionalidad muy buena de Spring Cloud Gateway, y es el uso de filtros, tanto los `Pre filtros` como los `Post filtros`,  estos son los filtros por los que pasará una request antes de acceder a los microservicios y después de accederlos respectivamente. Con esta imagen lo entenderás mejor:
![](/assets/images/Spring_Cloud_Gateway/Pasted image 20241017190108.png)
Spring Cloud Gateway viene con múltiples filtros por defecto que se pueden analizar con más detalle en la documentación. Además se pueden añadir filtros personalizados

### Implementación
1. Yendo al [initilizr](https://start.spring.io/) descargaremos un nuevo proyecto con las siguientes dependencias:
![](/assets/images/Spring_Cloud_Gateway/Pasted image 20241017190853.png)
La primera dependencia para indicar que queremos un `Gateway Reactivo`, y la segunda para poder descubrir el microservicio al que redirigir con`Eureka`.

2. Aquí un ejemplo de un posible `application.yml`

```application.yml
server:  
  port: 8072  
  
spring:  
  application:  
    name: "gatewayserver"  
  cloud:  
    gateway:  
      discovery:  
        locator:  
          enabled: true  

eureka:  
  instance:  
    preferIpAddress: true  
  client:  
    registerWithEureka: true  
    fetchRegistry: true  
    serviceUrl:  
      defaultZone: "http://localhost:8071/eureka/"  
```
Demos un vistazo rápido por este `application.yml`:
- `Cloud.gateway.discovery.locator.enabled`: Si lo seteamos a `true` no tenemos ni que crear rutas manualmente, ya a partir de aquí, podemos realizar peticiones al endpoint del microservicio que queramos(Qué esté registrado con Eureka obviamente): `http://localhost:8072/{NombreDelMicroservicio}/{endpoint}` 
-  Las propiedades de `eureka` son para poder registrar el GatewayServer y también para que pueda descubrir el resto de microservicios suscritos

Con estos simples 2 pasos ya podemos acceder a cualquier microservicio que esté suscrito en Eureka, ahora bien, ejecuta bien el endpoint para cada microservicio http://localhost:8072/{NombreDelMicroservicio}/{endpoint}




## Funcionalidades extra

La Funcionalidad principal de un gateway es tener un lugar fijo que reciba todas las peticiones para luego redirigirlas, pero tiene un potencial infinitamente mayor, veamos unas cuantas de las innumerables maneras por las que nos podemos beneficiar de un gateway server.


## Añadir seguridad
Obviamente el Gateway es el portero de todo el backend, así que necesita encargarse de la seguridad.

Cómo podemos añadir seguridad?

1. Creando una nueva clase `SecurityConfig` . La cual podemos crear dentro de una carpeta config
main
└── java
    └── com.kike.gatewayserver
        ├── config
        │   └── SecurityConfig
        └── GatewayserverApplication


2. En la clase `SecurityConfig.java` puedes incorporar el protocolo de seguridad que prefieras, en este ejemplo dejaremos pasar cualquier petición, pero recomiendo el protocolo OAuth2.0, ya que es una de las opciones más robustas y modernas.

```java
@Configuration  
@EnableWebFluxSecurity  
public class SecurityConfig {  
  
    @Bean  
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity serverHttpSecurity) {  
  
        serverHttpSecurity.authorizeExchange(exchange -> exchange.anyExchange().permitAll());  
  
        serverHttpSecurity.csrf(ServerHttpSecurity.CsrfSpec::disable);  
  
        return serverHttpSecurity.build();  
    }  
  
      
}
```


## Customizar endpoints
Anteriormente hemos visto que con la propiedad
```yml
spring:  
  cloud:  
    gateway:  
      discovery:  
        locator:  
          enabled: true  
```
Spring Gateway descubrirá automáticamente los posibles microservicios que esten suscritos a eureka.

Pero podemos personalizar estos endpoints.

1. Dentro de la clase principal del gateway: `GatewayserverApplication` crearemos un nuevo Bean para gestionar las rutas:
```java
@Bean  
RouteLocator myRouteConfig(RouteLocatorBuilder routeLocatorBuilder){  
    return routeLocatorBuilder.routes()  
            .route(p -> p  
                    .path("/myApp/event/**")  
                    .filters(f -> f.rewritePath("/myApp/event/(?.<segment>.*)","/${segment}"))  
                    .uri("lb://EVENT"))  
            .build();  
}
```
Desglosemos la expresión lambda
- `p.path()`: para especificar la ruta. Esto significa que el microservicio Events será accesible desde `"/myApp/event/**`
- `.filters()`: aquí se añaden los filtros discutidos anteriormente, es decir los post y pre filtros(sumados a los que nos vienen por defecto)
- `f -> f.rewritePath()`:  aquí le decimos que la petición recibida `"/myApp/event/{Endpoint}"`sea cambiada por `"/{Endpoint}"` ya que es lo que le enviará al microservicio. 
- A qué microservicio le enviará `"/{Endpoint}"` ? pues al especificado en el `.uri()`. Como vemos especificamos el microservicio `EVENT` suscrito en eureka. Y ya Spring Cloud se encargará de realizar el loadBalancing gracias a `Spring Cloud LoadBalancer`(El cual funciona por detrás)
De manera que para acceder al endpoint `api/create` de events, habrá que ejecutar el endpoint del gateway : `http://localhost:8072/myApp/event/api/create`
## Añadir cabeceras
En la función lambda de los filtros podemos añadir cabeceras:

```java
@Bean  
RouteLocator myRouteConfig(RouteLocatorBuilder routeLocatorBuilder){  
    return routeLocatorBuilder.routes()  
            .route(p -> p  
                    .path("/myApp/event/**")  
                    .filters(f -> f  
                            .rewritePath("/myApp/event/(?<segment>.*)","/${segment}")  
                            .addResponseHeader("X-Response-Time", LocalDateTime.now().toString()))  
                    .uri("lb://EVENT"))  
            .build();  
}
```
En este ejemplo la cabecera se añade en la respuesta, por lo que es un `post filtro`


## Reintentar Peticiones, CircuitBreaker, requestRateLimiter ...

Hay otros filtros muy poderosos que se pueden añadir
- El filtro de `retry`: Nos permite reintentar las peticiones a un microservicio si esta petición falla, para que no realize simplemente un intento
- El filtro de `circuit breaker pattern` denegar todas las peticiones en momentos en los que hay algún problema en el microservicio. En ciertas ocasiones es mejor denegar todas las peticiones a un microservicio para que este respire y pueda volver a estar operativo
- `requestRateLimiter`  nos permitirá limitar las peticiones que nos lleguen , quizás hay ciertos microservicios que no queremos que se congestionen y permitiremos un número de peticiones por minuto

Todos estos filtros los puedes aplicar a tu GatewayServer gracias a Resilient4j. La implementación y explicación en profundidad de cada uno de estos es para otro blog :)



