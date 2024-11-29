---
layout: single
title: Spring Cloud Stream
excerpt: "Spring Cloud Stream nos va a realizar toda la configuración necesaria para poder instaurar comunicación asíncrona entre nuestros servicios, preparando toda la configuración necesaria para que nosotros como desarrolladores nos encargemos simplemente de la lógica de negocio, y nos olvidemos de configurar topics(para Kafka) o queues(para RabbitMQ)"

date: 2024-11-12
classes: wide
header:
  teaser: /assets/images/Spring_Cloud_Stream/logo.gif
  teaser_home_page: true

categories:
  - Spring Cloud 
  - Spring Cloud Stream
  - Comunicación asíncrona
  - SpringBoot
tags:  
  - SpringBoot
---

## Introducción
La comunicación asíncrona es sin duda un  gran aliado de los microservicios. Si de una cosa pueden fardar las aplicaciones con microservicios frente a las aplicaciones monolíticas es del desacoplamiento que tienen estas, y este desacoplamiento también lo vemos en la comunicación asíncrona.

En el [último artículo](https://egarmar1.github.io/Kafka/#) expliqué como funciona Kafka y cómo permite que nuestros servicios se comuniquen asíncronamente. Si no sabes lo que es Kafka te recomiendo encarecidamente ver ese artículo para poder comprender los términos que se utilizen(eventos, producer, consumer ...)

Pero hoy veremos **cómo implementar** esta comunicación asíncrona con Spring Cloud Stream


## Spring Cloud Stream
Spring Cloud Stream va a hacer todo el trabajo de la conexión con Kafka, RabbitMQ o cualquier otra tecnología de mensajería que decidamos usar. De esta manera nosotros nos podemos encargar directamente de la lógica de negocio y el procesamiento de los eventos, olvidándonos de toda la configuración.



### Ejemplo
Considero que explicar la implementación al mismo tiempo que se muestra un ejemplo va a ser la mejor manera de interiorizar todo este maravillosos conocimiento.

- Imaginémonos una aplicación con dos microservicios, uno llamado `Noticias` encargado de todo lo relacionado a las noticias de una aplicación, y un segundo microservicio llamado `Mensajería` que se encarga de enviar emails,sms ... a los usuarios.

![](/assets/images/Spring_Cloud_Stream/Pasted image 20241107164111.png)
- De vez en cuando el microservicio `Noticias` quiere notificar a sus usuarios de una nueva noticia importante. Pero claro, quiere hacer esto de manera asíncrona, es decir, publicar la noticia y en un segundo plano avisar a los usuarios.
- Como hemos decidido usar kafka, tendremos un servidor kafka activo:
![](/assets/images/Spring_Cloud_Stream/Pasted image 20241107164932.png)

---

### Implementación en el microservicio **Mensajería**
El MS de **mensajería** quiere estar en escucha, para que cuando se cree una nueva noticia este se entere y envíe un correo a sus usuarios, para ello en el microservicio `Mensajería` :
1. Necesitaremos estas dependencias:
```java
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-stream</artifactId>  
</dependency>

<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>  
</dependency>  
<dependency> 
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-stream-test-binder</artifactId>
</dependency>

```
- La dependencia de `spring-cloud-stream` va a ser la que realize toda la configuración por detrás, tú utilizas canales(input y output) para los eventos, y te olvidas de topics, offsets, particiones ... 

- Por otra parte `spring-cloud-stream-binder-kafka` conecta Apache Kafka con esta 'abstracción' que hemos conseguido con la anterior dependencia, la cual nos permitiría no solo integrar kafka, sino, también otros  sistemas de mensajería como RabbitMQ(habría que usar la dependencia `spring-cloud-stream-binder-rabbit` para ese caso)
- `spring-cloud-stream-test-binder` simulará el funcionamiento de los binders(luego veremos qué son) para el testing.

2. Este microservicio podrá recibir los mensajes gracias a las **Functions** que proporciona Java. En este caso, estas funciones las podemos implementar en una carpeta que llamaremos `functions`, que ubicaremos en el paquete principal del microservicio(i.e `com.myApp.mensajeria.functions.MensajeríaFunctions`)

Concretamente crearemos una clase `MensajeriaFunctions`:

```java
package com.myApp.mensajeria.functions;  

import java.util.function.Consumer;  
  
@Configuration  
@AllArgsConstructor  
public class MensajeriaFunctions {  
  
    private static final Logger log = LoggerFactory.getLogger(MensajeriaFunctions.class);  
  
    @Bean  
    public Consumer<EmailDto> sendEmail() {  
        return emailDto -> {  
                log.info("sending Email to user " + emailDto.getRecipientEmail());  
                ...
        };  
    }  
}
```
- Contendrá la anotación `@Configuration`. E implementará una función **Consumer** la cual se encargará de recibir los detalles del email, para luego enviarlo al usuario(He quitado la implementación de enviar un email ya que se desvía del tema)

Vale bien, hemos creado la función, pero cómo sabe Spring Cloud a qué topic/queue conectarse para poder recibir esos eventos?

3. Para esto, en el application.yml necesitamos configurar un binding. **Un binding es la conexión entre la función y el topic de Kafka**:
```java
spring:  
  application:  
    name: "mensajeria"  
  cloud:  
    function:  
      definition: sendEmail
    stream:  
		bindings:  
		    sendEmail-in-0:  
				destination: send-email
		kafka:  
		  binder:  
		    brokers:  
		      - localhost:9092
```

Vayamos explicando cada parte:
- `spring.cloud.function.definition`: Aquí hemos añadido el nombre de la función que habíamos creado. Si queremos añadir otra función independiente las separaremos por un **;** `sendEmail;sendSMS`. Pero si quiseramos concatenar funciones, entonces tendríamos que usar el carácter **|**
- `spring.cloud.stream.bindings.sendEmail-in-0`:
	Para el nombre del binding, seguimos la convención [oficial](https://docs.spring.io/spring-cloud-stream/reference/spring-cloud-stream/functional-binding-names.html). Que consiste en poner en primer lugar el nombre de la función `sendEmail`, luego indicar `in` o `out` dependiendo de si es un consumer o un producer respectivamente, y por último indicar el índice, por si quisieramos que nuestra función se conectara a más topics(o queues para RabbitMQ).
- Por otra parte con la propiedad `spring.cloud.stream.bindings.sendEmail-in-0.destination` indicamos el nombre que le queremos dar al topic, en este caso **send-email**
- Y por último hay que indicar donde se encuentra nuestro servicio kafka corriendo, en mi caso en el puerto localhost:9092. Lo indicamos de la siguiente manera: ``spring.cloud.stream.kafka.binder.brokers.<Instancia Kafka>`

---
Hasta aquí ya hemos configurado totalmente el Consumer, es decir el microservicio que escucha los mensajes a través de un topic. Ahora nos queda por configurar el microservicio que envía los eventos(mensajes) al topic

### Implementación en el microservicio **Noticias**
Para este ejemplo el microservicio **noticias** quiere publicar en un topic los emails a los que quiere enviar una notificación sobre esta nueva noticia, para ello en el microservicio `Noticias` :

1. Añadiremos exactamente las mismas dependencias que hemos añadido en el microservicio **Mensajería**
2. Añadimos la configuración oportuna:
   ```java
   
   spring:  
  application:  
    name: "noticias"  
  cloud:  
    stream:  
      bindings:  
        sendEmail-out-0:  
          destination: send-email
      kafka:  
        binder:  
          brokers:  
            - localhost:9092
```

Es prácticamente la misma configuración que hemos hecho en el microservicio anterior (MS Mensajería), solo que el nombre del binding es **sendEmail-out-0**, es decir en vez de in hemos puesto out
Y ya, ya hemos preparado todo el entorno, solo queda ver cómo enviar el mensaje desde el consumer de manera asíncrona:


3. Lo único que hay que hacer es desde donde queramos enviar el mensaje ejecutar el siguiente comando :
```java
streamBridge.send("event-creation-trigger", emailDto);
```
StreamBridge se puede inyectar en tu clase con el Bean `StreamBridge`, este bean lo tenemos gracias a la dependencia `spring-cloud-stream` . Indicaremos el topic como primer argumento, y como segundo argumento el dto del email que recibirá  el microservicio `mensajeria`.

