---
layout: single
title: Spring Cloud Stream
excerpt: "Denegar todas las peticiones de un microservicio durante un periodo de tiempo puede ser muy buena idea si este está teniendo problemas de conexión, o congestionado debido a que recibe un gran numero de peticiones. 
El **Circuit Breaker Pattern** es una buena solución para gestionar estos escenarios."

date: 2024-11-22
classes: wide
header:
  teaser: /assets/images/CircuitBreaker/Pasted image 20241203162831.png
  teaser_home_page: true

categories:
  - Spring Cloud 
  - Spring Cloud Gateway
  - Circuit Breaker Pattern
  - SpringBoot
  - Resilience4j
tags:  
  - SpringBoot
---

## Introducción
Denegar todas las peticiones de un microservicio durante un periodo de tiempo puede ser muy buena idea si este está teniendo problemas de conexión, o congestionado debido a que recibe un gran numero de peticiones. 

El **Circuit Breaker Pattern** es una buena solución para gestionar estos escenarios, dando resiliencia a nuestro proyecto. Este patrón lo integraremos gracias a la herramienta Resilience4j(La más popular y recomendada).


## Qué es el patrón circuit breaker 

La mejor manera para que nunca se te olvide en qué consiste este patrón es entendiendo su significado.

Todos tenemos un `circuit breaker`(disyuntor) en nuestra casa. Este circuit breaker se encarga de frenar los problemas que puedan ocurrir en el sistema eléctrico. Como por ejemplo, que se use una potencia eléctrica por encima de lo permitido, o que haya un cortocircuito.

Pues el circuit breaker pattern funciona igual pero en microservicios.


### Cómo funciona este patrón
El **Circuit breaker pattern** monitorizará las peticiones que se dirijan a un microservicio.

Por una parte si este microservicio tarda demasiado en responder, entonces el circuit breaker acabará con esa request.

Por otra parte si un porcentaje específico(threshold) de las respuestas del microservicio tardan demasiado, entonces el circuit breaker saltará, denegando todas las requests futuras, devolviendo a cada request un mensaje indicando que el microservicio no está disponible.

### Estados del patrón
El patrón `circuit breaker` tiene 3 estados:

- **Closed**: Este es el estado inicial, y como indica la palabra `closed`, aquí el circuit breaker está cerrado, permitiendo la entrada de todos las requests al microservicio
- **Open**: A este estado se entra cuando se ha superado el threshold (procentaje límite)  de peticiones falladas. Y se denegarán todas las peticiones duante un tiempo determinado
- **Half-Open**: Después de estar un periodo de tiempo en el estado `Open` se entrará en el estado `Half-Open`, permitiendo la entrada de unas pocas requests, y si la cosa ha mejorado(es decir, que el microservicio vuelve a funcionar) entonces pasará al estado `Open`, si no, volverá al estado `Closed`
![](/assets/images/CircuitBreaker/Pasted image 20241203162831.png)

Con esta imagen se muestra cómo funcionaría el flujo de los estados.

Y probablemente surjan dudas como: "Oye, ¿Y cuándo exactamente se pasa al estado _open_? o, ¿Cómo se decide si se pasa al estado _closed_ o al estado _half-open_?"

Todo esto se decide gracias al threshold que especificaremos al implementar este patrón. 
El **Threshold** no es más que un porcentaje límite, y si ese porcentaje límite se sobrepasa, entonces pasaremos al estado correspondiente.

Por ejemplo podemos indicar un threshold del 50%, por lo que si el 50% de las requests fallan, se pasará al estado open. Por otra parte también se puede especificar el threshold del estado Half-Open. Cómo se implementa lo veremos a continuación


## Implementación del patrón circuit breaker en Spring Boot
Qué mejor manera de comprender este patrón que mostrándolo en un proyecto real.


![](/assets/images/CircuitBreaker/Pasted image 20241203171917.png)
Imagina un proyecto con un **Gateway Server**([Spring Cloud Gateway](https://egarmar1.github.io/Spring-Cloud-Gateway/)) y diferentes microservicios, en este proyecto queremos añadir el patrón **circuit breaker** al microservicio `Events Reviews`.

En nuestro ejemplo solo habrá que realizar cambios en el Gateway Server ya que es el que se  encarga de todo el control de las peticiones.

Dicho esto haremos las siguientes modificaciones en el **Gateway Server**

1. Añadir las siguientes dependencias:

```java
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>  
</dependency>
<dependency>  
    <groupId>io.github.resilience4j</groupId>  
    <artifactId>resilience4j-reactor</artifactId>  
</dependency>  

```
- El **starter** lo podemos obtener del [initialzr](https://start.spring.io/), lo que integrará la funcionalidad de Resilience4j con **Spring Cloud Circuit Breaker**, ya que este último es un marco genérico que necesita una implementación y utilizará Resilience4j como tal.

- Por otra parte está la dependencia de `resilience4j-reactor` que nos va a permitir aplicar patrones de resiliencia que `Resilience4j` nos proporciona para flujos reactivos.

2. A continuación tenemos que añadir un filtro a la ruta que redirige al **microservicio Events Reviews**. Si no conoces que es esto de los filtros de Spring Cloud Gateway te recomiendo encarecidamente pasarte por mi otro artículo donde explico en detalle [Spring Cloud Gateway](https://egarmar1.github.io/Spring-Cloud-Gateway/).
```java
  
@SpringBootApplication  
public class GatewayserverApplication {  
    public static void main(String[] args) {  
        SpringApplication.run(GatewayserverApplication.class, args);  
    }  
    @Bean  
    public RouteLocator fastBookRouteConfig(RouteLocatorBuilder routeLocatorBuilder) {  
        return routeLocatorBuilder.routes()  
                .route(p -> p  
                        .path("/fastbook/eventsReviews/**")  
                        .filters(f -> f  
                                .rewritePath("/fastbook/eventsReviews/(?<segment>.*)", "/${segment}")  
                                .circuitBreaker(config -> config.setName("eventsReviewsBreaker")))  
                        .uri("lb://EVENTREVIEWS"))  
                .build();  
    }  
}
```

- Como se puede observar solo hay que añadir el filtro `circuitBreaker(config -> config.setName("eventsReviewsBreaker"))`  y si lo deseamos también podemos setear el nombre.

3. Además es necesario configurar este **circuit breaker**, esta configuración se realizará en el `application.yml`:
```application.yml
resilience4j.circuitbreaker:  
  configs:  
    default:  
      slidingWindowSize: 10  
      permittedNumberOfCallsInHalfOpenState: 2  
      failureRateThreshold: 50  
      waitDurationInOpenState: 10000
```
Es importante entender para qué sirve cada configuración:
- **SlidingWindowSize**: Se refiere al número de llamadas que serán evaluadas en el estado `Closed`. En este evaluará 10 requests
- **permittedNumberOfCallsInHalfOpenState**: Número de llamadas evaluadas al entrar en el estado `Half-Open`. Por lo que cuando se entre en el estado `Half-Open` se permitirán 2 requests, y se realizará la evaluación de estas 2 llamadas
- **failureRateThreshold**: Aquí especificaremos el threshold, es decir el límite de llamadas fallidas. En nuestro caso haremos saltar el circuit breaker, en el momento que el 50% de las requests fallen.
- **waitDurationInOpenState**: Por último como el nombre indica esta propiedad muestra el tiempo en el estado `Open` antes de pasar al estado `Half-Open`

Me gustaría puntualizar lo siguiente, y es que el circuit breaker no salta  hasta que el buffer especificado se haya llenado, por ejemplo, si nosotros hemos configurado un **slidingWindowsSize** de 10, hasta que no se hayan recibido 10 peticiones no se hará la evaluación, al igual con el **permittedNumberOfCallsInHalfOpenState**, que en nuestro ejemplo necesitará 2 llamadas antes de que pueda evaluar si saltar o no.

4. Por último creo conveniente añadir una opción adicional al filtro de **circuit breaker**:
```java
@SpringBootApplication  
public class GatewayserverApplication {  
    public static void main(String[] args) {  
        SpringApplication.run(GatewayserverApplication.class, args);  
    }  
    @Bean  
    public RouteLocator fastBookRouteConfig(RouteLocatorBuilder routeLocatorBuilder) {  
        return routeLocatorBuilder.routes()  
            .route(p -> p  
		        .path("/fastbook/eventsReviews/**")  
		        .filters(f -> f.rewritePath("/fastbook/eventsReviews/(?<segment>.*)", "/${segment}")  
	                .circuitBreaker(config -> config  
                        .setName("eventsReviewsBreaker")  
                        .setFallbackUri("forward:/fallback/contactSupport")))  
		        .uri("lb://EVENTREVIEWS"))  
			.build();
    }  
}
```
Esta opción añadida `.setFallbackUri("forward:/fallback/contactSupport")` redirigirá al endpoint `/fallback/contactSupport` en el momento que se reciba una request y el circuit breaker esté activado, es decir, que se encuentre en el estado `Open`.


Por simplicidad, este endpoint lo único que hará es devolver un mensaje:
```java
  
@RestController  
@RequestMapping("/fallback")  
public class FallbackController {  

	@RequestMapping("/contactSupport")  
	public ResponseEntity<String> contactSupport() {  
	    String message = "An error occurred. Please try again later or contact the support team.";  
	    return ResponseEntity  
	            .status(503) 
	            .body(message);  
	} 
}
```
Pero en un proyecto real se podrían realizar acciones como por ejemplo avisar al equipo técnico.

### Circuit Breaker en acción

Ahora toca verlo en acción, y comprobar que funciona como esperábamos.

Recordemos el proyecto, un Gateway Server que redirige las peticiones, y un Microservicio `Events Reviews Microservice`  encargado de todo lo relacionado a las reviews de los eventos deportivos

![](/assets/images/CircuitBreaker/Pasted image 20241203171917.png)

Para poder enseñar de la manera más simple el funcionamiento de este patrón, en el microservicio `Events Reviews` crearemos dos endpoints:

```java
@PostMapping("/create")  
public ResponseEntity<ResponseDto> createEventReview(@RequestBody @Valid EventReviewsCreateDto eventReviewsCreateDto) {  
    try {  
        Thread.sleep(5000);  
    } catch (InterruptedException e) {  
        throw new RuntimeException(e);  
    }  
  
    return ResponseEntity.status(201).body(new ResponseDto(STATUS_201, MESSAGE_201));  
}  
  
@GetMapping("/doNothing")  
public ResponseEntity<ResponseDto> doNothing() {  
  
    return ResponseEntity.status(200).body(new ResponseDto(STATUS_200, MESSAGE_200));  
}
```
- El endpoint **/create** tendrá una espera de 5 segundos(por lo que fallará)
- El endpoint **/doNothing** devolverá un mensaje exitoso

Puntualizar que estos endpoints no tienen ninguna funcionalidad ni lógica de negocio, con lo que nos tenemos que quedar es que **create** tardará demasiado en responder por lo que fallará, y **/doNothing** funcionará correctamente


#### Ejecutar peticiones
Realizaremos las peticiones con postman.

- Si intentamos enviar una petición al endpoint **/create** el circuit breaker nos devolverá el mensaje del fallback:
![](/assets/images/CircuitBreaker/Pasted image 20241205162218.png)
Esto ha ocurrido ya que este endpoint tiene un sleep, que hace que se sobrepase el tiempo permitido de respuesta.

- En cambio, si enviamos una petición al endpoint **/doNothing** se nos devolverá una respuesta exitosa:
![](/assets/images/CircuitBreaker/Pasted image 20241205162830.png)

Pero, ¿Cómo podemos ver el estado en el que se encuentra? ¿Cómo se yo como desarrollador si estamos en el estado closed, open, o half-open? o ¿Cúantas han fallado?

#### Info del circuitBreaker
Toda esa información la podemos obtener accediendo al endpoint **/actuator/circuitbreakers** del gateway server.
![](/assets/images/CircuitBreaker/Pasted image 20241205163633.png)

En un Json obtenemos toda la información de los **circuit Breakers**, de hecho en la propiedad `bufferedCalls` observamos que hemos realizado 2 llamadas, y luego en `failedCalls` se ve que una de ellas ha fallado( la realizada contra en endpoint `/create`). Por último el `state` está en `CLOSED`.

---

#### Escenario de ejemplo
1. Vamos a hacer que el estado pase de `CLOSED` a `OPEN`, para ello realizaremos 10 llamadas al endpoint `/create` el cual siempre falla.

Al realizar estas llamadas, el state pasará al estado `OPEN` , y durante 10 segundos redirigirá todas las peticiones(tanto correctas como incorrectas) al `fallback endpoint` del gateway, sin permitir que lleguen al microservicio
![](/assets/images/CircuitBreaker/Pasted image 20241205164336.png)
Como vemos hemos llegado a 10 bufferedCalls, de las cuales 9 han fallado, por lo que se ha pasado al estado `OPEN`

2. Después de 10 segundos(especificados antes en la configuración), vemos que se puede volver a enviar una request y recibir una respuesta exitosa
![](/assets/images/CircuitBreaker/Pasted image 20241205162830.png)
Esto es porque nos encontramos en el estado `HALF-OPEN`:
![](/assets/images/CircuitBreaker/Pasted image 20241205164649.png)
3. En este estado permitiremos 2 llamadas, y si de esas 2 el 50% fallan entonces volvemos al estado `OPEN` denegando todas las requests durante otros 10 segundos, y si no, pasamos al estado CLOSED permitiendo de nuevo todas las requests.(Todos esos números los hemos configurado anteriormente).

(Si no sabes lo que es actuator por favor mira algún tutorial, es necesario que cubras esta característica OBLIGATORIA de Spring Boot )


### Conclusión
Antes de terminar quiero que sepas que este patrón puede aumentar de manera considerable el rendimiento. En ciertas situaciones la congestión de un microservicio puede no suponer un gran problema, pero en otras puede afectar a múltiples microservicios a la vez(ya que se realizan llamadas entre estos).
Y no solo afecta a los microservicios directamente relacionados. El resto de microservicios también se verán afectados, ya que la congestión puede saturar las colas de eventos o conexiones, afectando al resto de peticiones

Hasta aquí el tutorial del **Circuit Breaker Pattern**, espero que te haya servido para comprender el patrón. Nos vemos !!