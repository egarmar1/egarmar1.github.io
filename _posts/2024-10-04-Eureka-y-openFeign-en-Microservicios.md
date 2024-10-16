---
layout: single
title: Eureka y openFeign para la comunicación entre Microservicios
excerpt: "Kubernetes se ha convertido en el rey de la gestión de microservicios, pero no te creas que es siempre conveniente.

Por ejemplo, Kubernetes requiere muchos recursos para poder funcionar, cosa que no cualquier empresa se puede permitir.

Y no vengo a decir que Kubernetes no sea bueno, no me malinterpreteis, solo digo que no es la mejor opción en el 100% de las ocasiones.

A partir de aquí sale la duda de, sin kubernetes... ¿Cómo puedo hacer para que mis servicios se comuniquen? ¿Cómo se encuentran entre ellos? y ¿Cómo funciona el LoadBalancer aquí? 

La respuesta rápida es **Eureka** y **OpenFeign**."
date: 2024-10-03
classes: wide
header:
  teaser: /assets/images/Eureka/eurekaIcon.png
  teaser_home_page: true
  icon: /assets/images/eurekaIcon.png
categories:
  - Communication
  - SpringBoot
tags:  
  - OpenFeign
  - Eureka
  - SpringBoot
---



## Eureka y OpenFeign

**Eureka** será la encargada de saber qué microservicios están disponibles, los registrará y de esta manera un microservicio podrá preguntar a Eureka y localizar el resto de microservicios

**OpenFeign** nos va a permitir realizar peticiones entre microservicios de una forma muy sencilla. Además permitirá que cada cliente tenga su propio LoadBalancer

## Estructura

### Microservicios
Para comprender el concepto veremos un ejemplo de proyecto son solo 2 microservicios

![](/assets/images/Eureka/Pasted image 20241003142046.png)

El primer microservicio es **Events**, que contendrá la información de eventos de una aplicación en la que la gente se puede apuntar a diferentes eventos(deportivos, de hobbies...)

Por otra parte **Bookings** contendrá las reservas realizadas por los usuarios, los cuales pueden reservar un sitio para un evento


### Situación
Pongámonos en la situación de que un usuario quiere reservar una plaza para un evento, al crear una reserva(booking) necesitaremos avisar al microservicio Events para que pueda tener un conteo del número de personas que se han apuntado



## Descubrimiento con Eureka

## Crear servidor Eureka

En primer lugar será necesario crear un nuevo proyecto con la dependencia de ==Eureka Server==

![](/assets/images/Eureka/Pasted image 20241003143031.png)

---

Luego, en el `application.yml` de este proyecto habrá que especificar la url donde se encontrará este servidor Eureka

```yml
server:  
  port: 8071  
  
spring:  
  application:  
    name: "eurekaserver"  

eureka:  
  instance:  
    hostname: localhost  
  client:  
    fetchRegistry: false  
    registerWithEureka: false  
    serviceUrl:  
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```
- `fetchRegistry`: Si se activa entonces se obtendrá el registro de otros servicios(cosa que no le interesa, solo le interesa a los microservicios que se quieran comunicar)
- `registerWithEureka`: Al activarse, la instancia se registrará en el servidor de eureka, y no tiene sentido registrar eureka en el propio eureka.
- `serviceUrl.defaultZone`: Por último especificamos la url donde se encontrará el servidor Eureka.
---


Por último añadimos la anotación `@EnableEurekaServer` en la clase principal del proyecto.

```java
@SpringBootApplication  
@EnableEurekaServer  
public class EurekaserverApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(EurekaserverApplication.class, args);  
    }  
  
}
```

### Resultado
Accediendo a `http://localhost:8071/` ya podemos ver Eureka funcionando:

![](/assets/images/Eureka/Pasted image 20241003144642.png)

Y obviamente aún no tenemos ninguna instancia registrada.

## Conectar Microservicios con Eureka
Habiendo terminado con el servidor **Eureka**, ahora seguimos con los microservicios, los cuales se tienen que registrar en el servidor Eureka.


En primer lugar, los microservicios(en este caso `Events` y `Bookings`) requieren de la dependencia **eureka client**:

Así que la añadimos al `pom.xml` :
```xml
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>  
</dependency>  
```

Eso sí, también es necesario añadir el dependencyManagement:
```xml
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>${spring-cloud.version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

El `spring-cloud.version` se especifica en las properties: 
```xml
<spring-cloud.version>2023.0.3</spring-cloud.version>
```

En caso de cualquier duda, ve al initilizr, elige la dependencia, clicka en `Explore`, y a partir de aquí copia la dependencia, el dependencyManagement y el spring-cloud.version como acabamos de ver.



---
En segundo lugar, cada microservicio necesita saber donde se encuentra el servidor Eureka, por lo que habrá que señalarlo en el `application.yml`:

```yml
server:  
  port: 8090  
  
spring:  
  application:  
    name: "booking"
    
eureka:  
  instance:  
    preferIpAddress: true  
  client:  
    fetchRegistry: true  
    registerWithEureka: true  
    serviceUrl:  
      defaultZone: http://localhost:8071/eureka/
```
Realizaremos lo mismo en el microservicio **Events** 

### Resultado
Ya está toda la configuración del descubrimiento terminada, a partir de aquí cada microservicio puede ser conocedor del paradero de otro microservicio gracias a Eureka!!

De hecho, podemos ver que al iniciar los microservicios, estos se enceuntran registrados en Eureka:

![](/assets/images/Eureka/Pasted image 20241003165012.png)


## OpenFeign
Ahora `Bookings` ya sabe donde se encuentra`Events`, solo queda que se comunique con él.

Aquí es cuando entra **OpenFeign**

Y solo necesitamos configurar OpenFeign en el microservicio que realiza la petición(Bookings), el que recibe la petición(Events) no requiere ningún cambio
### Configuración
El microservicio `Bookings` quiere avisar a `Events` de que va crear una nueva reserva, pero quiere avisarle justo antes de crear esta reserva para el evento


En resumen, Bookings quiere ejecutar un método de `Events` concretamente el que se encarga de aumentar en una unidad la cuenta de reservas de un evento:
`EventController.java`:
```java
@PutMapping("/update/currentBookingsCount")  
public ResponseEntity<ResponseDto> updateCurrentBookingsCount(@Valid @RequestParam Long eventId) {  
  
    boolean isUpdated = iEventService.increaseCurrentBookings(eventId);  
  
    if (isUpdated)  
        return ResponseEntity  
                .status(OK)  
                .body(new ResponseDto(STATUS_200, MESSAGE_200));  
    else  
        return ResponseEntity  
                .status(EXPECTATION_FAILED)  
                .body(new ResponseDto(STATUS_417, MESSAGE_417_UPDATE));  
}
```


Vale, pero ¿Cómo puede el MS Bookings ejecutar ese endpoint? 
- En primer lugar hay que añadir la dependencia de openFeign.
- Y en segundo lugar añadir la anotación @FeignClient encima de la clase principal del microservicio.

```xml
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-openfeign</artifactId>  
</dependency>
```

Y para que `Bookings` pueda ejecutar ese endpoint del microservicio `Events` tiene que crear una interfaz `FeignClient`, esta puede ser creada dentro de la carpeta `services`, y la llamaremos `EventsFeignClient`

```java
@FeignClient(name = "event")  
public interface EventsFeignClient {  
  
    @PutMapping(value = "/api/update/currentBookingsCount")  
    public ResponseEntity<ResponseDto> updateCurrentBookingsCount(@RequestParam Long eventId);  
}
```

No hace falta hacer nada más, ya que `FeignClient` tiene un enfoque declarativo, al igual que `JPA`, es decir, que por detrás genera automáticamente el código necesario para realizar la petición. Nosotros le indicamos solamente el nombre del servicio(el que está suscrito en eureka) y el método del controlador que queremos ejecutar


---
Así es como conseguimos comunicarnos con el microservicio `Events`, y lo podemos hacer desde cualquier parte del código, solo hay que inyectar la dependencia(El Bean) y realizar la llamada:


```java
@Override  
public void createBooking(BookingDto bookingDto) {  
  
    Optional<Booking> existingBooking = bookingRepository.findByUserIdAndServiceId(bookingDto.getUserId(), bookingDto.getEventId());  
  
    if (existingBooking.isPresent())  
        throw new BookingAlreadyExistsException("The book with serviceId: " +  
                bookingDto.getEventId() + " and userId: " +  
                bookingDto.getUserId() + " already exists");  
  
    Booking booking = BookingMapper.mapToBooking(bookingDto, new Booking());  
    ResponseEntity<ResponseDto> responseDtoResponseEntity = eventsFeignClient.updateCurrentBookingsCount(bookingDto.getEventId());  
  
    if (responseDtoResponseEntity.getStatusCode() == HttpStatus.OK)  
        bookingRepository.save(booking);  
  
}
```
De este modo, solo se creará la reserva si hay espacio suficiente para ese evento y si el microservicio Events nos devuelve un HTTP Status = OK (200)