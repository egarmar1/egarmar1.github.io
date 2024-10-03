---
layout: single
title: Documentar APIs en Spring Boot con OpenAPI
excerpt: "¡Qué pereza!. Esta sea probablemente la primera frase que te llegue a la cabeza cuando pienses en documentar. Pero de verdad, confía en mí cuando te digo que la documentación de APIs en Spring Boot es muy sencilla, y es sencilla gracias a OpenAPI."
date: 2024-10-03
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/openAPIImage.png
  teaser_home_page: true
  icon: /assets/images/openAPIImage.png
categories:
  - documentation
  - SpringBoot
tags:  
  - documentation
  - SpringBoot
---


## Introducción
¡Qué pereza!. Esta sea probablemente la primera frase que te llegue a la cabeza cuando pienses en documentar.

Pero de verdad, confía en mí cuando te digo que la documentación de APIs en Spring Boot es muy sencilla, y es sencilla gracias a OpenAPI.

## Entorno de Ejemplo

Vamos a ver el ejemplo de una API muy simple, una API para poder crear eventos deportivos en una aplicación

Este sería el controller:
```java
package com.kike.events.controller;  
  
@RestController  
@RequestMapping(value = "/api",consumes = MediaType.APPLICATION_JSON_VALUE)  
@AllArgsConstructor  
public class EventController {  
  
    IEventService iEventService;  
  
    @PostMapping("/create")  
    public ResponseEntity<ResponseDto> createEvent(@Valid @RequestBody EventDto eventDto){  
  
        iEventService.createEvent(eventDto);  
  
        return ResponseEntity  
                .status(CREATED)  
                .body(new ResponseDto(STATUS_200,MESSAGE_201));  
    }  
}
```

Un controlador sencillo, con una única API para poder crear un evento enviando la información de este mismo.
La información del evento se enviaría en un post con el siguiente dto:
```java
package com.kike.events.dto;  
  
import java.math.BigDecimal;  
  
@Data  
public class EventDto {  
  
    @NotEmpty(message = "Title cannot be empty")  
    private String title;  
  
    @NotEmpty(message = "Description cannot be empty")  
    private String description;  
  
    @PositiveOrZero(message = "Price must be greater or equal than zero")  
    private BigDecimal price;  
  
    @Positive(message = "Price must be greater or equal than zero")  
    private int vendorId;  
  
    private Availability availability;  
}
```

Por otra parte la API nos responderá con une mensaje exitoso, este mensaje tiene la estructura del ResponseDto que hay a continuación:
```java
package com.kike.events.dto;    
  
@Data  
@AllArgsConstructor  
public class ResponseDto {  
  
    private String statusCode;  
    private String statusMsg;  
}
```

Ya está, esa es toda la estructura, la suficiente que nos permitirá entender a la perfección la documentación con OpenAPI


## Pasos previos
Primero de todo, necesitamos añadir al **pom.xml** el starter de openapi, el cual puedes encontrar en [maven repository](https://mvnrepository.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui): 
```xml
<dependency>  
    <groupId>org.springdoc</groupId>  
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>  
    <version>2.6.0</version>  
</dependency>
```



## Primera vista
Y aunque no lo creas, solo añadiendo el starter ya tienes en cierta medida la documentación hecha, y no me tienes que creer, tan solo accede o a la url:
`http://localhost:8080/swagger-ui/index.html#/`(localhost:8080 ya que mi microservicio se encuentra ahí)

En esta url encontrarás tu api documentada:
<img src="../assets/Pasted image 20240930160112.png">

Vemos que aparecen nuestros dtos, nuestro controlador, nuestra api para crear eventos ...

Asimismo, podemos ejecutar un request con la estructura que habíamos indicado anteriormente
<img src="../assets/Pasted image 20240930160807.png">



No obstante, esta documentación es muy pobre, piensa que tiene que ser lo más clara posible, de modo que alguien que no tenga ni idea del back tenga  información explicativa para crear un evento sin complicación alguna.






## Mejorando la documentación

### Documentando Aplicación principal
Vayamos de arriba a abajo, comenzemos documentando la propia aplicación:

```java
package com.kike.events;  
  
import io.swagger.v3.oas.annotations.*;
@SpringBootApplication  
@OpenAPIDefinition(  
       info = @Info(  
             title = "Sport Events microservice REST API Documentation",  
             description = "Sport Events microservice REST API Documentation",  
             version = "v1",  
             contact = @Contact(  
                   name = "Enrique Garcia",  
                   email = "kike@gmail.com",  
                   url = "https://kikegm.com"  
             ),  
             license = @License(  
                   name = "Apache 2.0",  
                   url = "https://kikegm.com"  
             )  
       ),  
       externalDocs = @ExternalDocumentation(  
             description = "Sport Events microservice REST API Documentation",  
             url = "https://otherurl.com"  
  
       )  
)public class EventsApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(EventsApplication.class, args);  
    }  
  
}
```

Encima de la clase principal de la aplicación Spring Boot incluiremos la anotación `@OpenAPIDefinition`, y dentro de esta anotación añadiremos:
- **info** : información general de la aplicación entera, con título, descripción, versión, contacto para cualquier duda sobre la documentación, licencia...
- **externalDocs**: En caso de que haya más documentación externa se puede añadir

#### Resultado
¿Y cómo se ve esto? De la siguiente manera:
<img src="../assets/Pasted image 20240930161831.png">

Así es como conseguimos incluir información general de la aplicación, con datos de contacto, licencia ...


### Documentando el controlador

#### Documentar clase
`Event-controller` no es un nombre muy correcto para poner en una documentación, además, ¿De qué se encarga este controlador?

Incorporemos la anotación `@Tag` encima de la clase del controlador:
```java
@Tag(  
        name = "CRUD REST API for events in SportEvents",  
        description = "CRUD REST APIs for events in SportEvents to CREATE," +  
                " UPDATE, FETCH AND DELETE events "  
)  
@RestController  
@RequestMapping(value = "/api", produces = MediaType.APPLICATION_JSON_VALUE)  
@AllArgsConstructor  
public class EventController {
...
}
```

Así indicamos que este controlador es para realizar operaciones  CRUD(Create, Read, Update, Delete) aunque por ahora solo se puede crear... pero bueno, la idea es que acabe siendo un controlador CRUD


#### Resultado
<img src="../assets/Pasted image 20240930162739.png">

Vale bien, ahora ya se sabe para que sirve ese controlador


#### Documentar APIs
Pasemos a la parte más interesante, documentar las APIs:
Encima de cada método tendremos que añadir la anotación `@Operation`:
```java
@Operation(  
        summary = "Create event REST API",  
        description = "REST API to create an event in the app SportEvents ",  
        responses = {  
                @ApiResponse(  
                        responseCode = "201",  
                        description = "HTTP Status CREATED"  
                ),  
                @ApiResponse(  
                        responseCode = "400",  
                        description = "HTTP Status BAD REQUEST"  
                ),  
                @ApiResponse(  
                        responseCode = "500",  
                        description = "HTTP Status INTERNAL SERVER ERROR",  
                        content = @Content(  
                                schema = @Schema(implementation = ErrorResponseDto.class)  
                        )  
                )  
        }  
  
)
```
Aquí tenemos:
- **summary y description**: Para dar un título y descripción sobre el endpoint
- **responses**: Aquí señalaremos todas las posibles respuestas que pueda devolver esta api, en este caso hay 3 posibles respuestas anotadas con `@ApiResponse`. Además se puede mostar con `@Content` el esquema que se devolverá.

El esquema es un dto, el cual también se puede documentar:
#### Documentar Dtos

```java
package com.kike.events.dto;  
  
import io.swagger.v3.oas.annotations.media.Schema;  
  
@Data  
@AllArgsConstructor  
@Schema(  
        name = "ErrorResponse",  
        description = "Schema to hold error response information"  
)  
public class ErrorResponseDto {  
    @Schema(  
            description = "API path invoked by client"  
    )  
    private String apiPath; // The api path that failed  
  
    @Schema(  
            description = "Error code representing the error happened"  
    )  
    private HttpStatus errorCode;  
  
    @Schema(  
            description = "Error message representing the error happened"  
    )  
    private String errorMessage;  
  
    @Schema(  
            description = "Time representing when the error happened"  
    )  
    private LocalDateTime errorTime;  
}
```
Y no solo documentes las respuestas de error, si no también el resto de dtos, como por ejemplo el EventsDto para que quien llame a la api sepa ejemplos concretos:

```java
@Schema(  
        name = "Event",  
        description = "Schema to hold the event information"  
)  
@Data  
public class EventDto {  
  
  
    @Schema(description = "Title of the Event ", example = "Football Class in center of Valencia")  
    private String title;  
  
    @Schema(  
            description = "Title of the Event ", example = "Football Class in center of Valencia, on day 25 of october, no minimum level required "  
    )  
    private String description;  
  
    @Schema(  
            description = "Price for signing up ", example = "10.9"  
    )  
    private BigDecimal price;  
  
    @Schema(  
            description = "Unique identifier for vendor", example = "154"  
    )  
    private int vendorId;  
  
    @Schema(  
            description = "Availability of the event, it shows wether is posible to book a place or not", example = "AVAILABLE"  
    )  
    private Availability availability;  
  
}
```


#### Resultado final
Ahora sí.
Hemos conseguido una api totalmente documentada, con un ejemplo para ejecutar:
<img src="../assets/Pasted image 20240930164209.png">

Con los distintos tipos de posibles respuestas:
<img src="../assets/Pasted image 20240930164245.png">

Y con los dtos explicados a detalle:
<img src="../assets/Pasted image 20240930164357.png">
