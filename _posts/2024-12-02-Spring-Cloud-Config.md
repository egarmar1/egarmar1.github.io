---
layout: single
title: Spring Cloud Config Server con Github
excerpt: "En una aplicación con una arquitectura de microservicios realizar todas las configuraciones en cada uno de los microservicios por separado puede ser poco conveniente, inseguro y mucho menos escalable. Déjame que te lo demuestre.
"

date: 2024-12-02
classes: wide
header:
  teaser: /assets/images/Spring_Cloud_Config/Pasted image 20241216163110.png
  teaser_home_page: true

categories:
  - Spring Cloud 
  - Spring Cloud Config
  - Spring Cloud Bus
  - SpringBoot
tags:  
  - SpringBoot
---

## Introducción
En una aplicación con una arquitectura de microservicios realizar todas las configuraciones en cada uno de los microservicios por separado puede ser poco conveniente, inseguro y mucho menos escalable. Déjame que te lo demuestre.


### Proyecto sin Spring Cloud Config Server
Primero mostremos la situación de un proyecto sin Spring Cloud Config, para luego mostrar el proyecto con Spring Cloud Config Server y observar cómo nos ayudará.

- En este microservicio tenemos un `resources/application.yml`:

```yml
server:  
  port: 8030  
  
spring:  
  application:  
    name: "eventReviews"  
  config:  
    import:  
      - "application_prod.yml"  
      - "application_qa.yml"  
  profiles:  
    active:  
      "qa"  
```

Este application.yml se extiende con otro application.yml dependiendo de si es un entorno de **qa**(`application_qa.yml`) o un entorno de **producción**(`application_prod.yml`). Ahora como vemos en `spring.profiles.active=qa` hemos añadido el application_qa.yml.

Esto no es conveniente, vemos varios problemas:

1. En primer lugar tenemos que tener todos los application.yml distribuidos en microservicios, en un proyecto real habrán decenas o cientos de microservicios, y cada uno de ellos con sus propios application.yml. Sería mucho más cómodo tener los archivos de configuración centralizados.

2. En segundo lugar habrán ciertas propiedades que se repitan en diferentes application.yml, por ejemplo, configuraciones sensibles, o contraseñas. Además hay más probabilidad de errores de seguridad si tenemos que gestionar los secretos en múltiples microservicios.

4. Otro punto es el auditing y versioning. De la manera que lo estamos haciendo no podemos dejar huella de otras versiones de configuraciones. Estaría muy bien si pudieramos conetarlo con algun sistema de control de versiones como GIT(`Spoiler `con Spring Cloud Config se puede )
5. Por último, si queremos cambiar alguna propiedad de un archivo de configuración tenemos que reiniciar el microservicio, por lo tanto todas las instancias de este.


### Spring Cloud Config Server
Considero suficiente la demostración de los problemas de no usar un sistema centralizado para colocar todas las configuraciones(aunque hay más).

![](/assets/images/Spring_Cloud_Config/Pasted image 20241216163110.png)


Este es el esquema del proyecto,`Spring Cloud Config` por su parte nos dará este entorno centralizado.
Como vemos en el esquema, tendremos:
- Los **microservicios** por una parte
- El Servidor **Spring Cloud Config**
- Un **Sistema de control de versiones** donde guardaremos los archivos de configuración. También es posible guardar los archivos localmente o en el classpath del config server(aunque pierder el auditing, versioning, seguridad ...)

Vamos a ir configurándolo paso a paso para comprender de verdad cómo resuelve cada uno de los incovenientes mencionados anteriormente


#### Crear el Config Server
1. Lo primero de todo es crear un nuevo proyecto Spring Boot en [initializr](https://start.spring.io/) con un par de dependiencias:
- La primera es **Config Server**.
![](/assets/images/Spring_Cloud_Config/Pasted image 20241216163419.png) 
- La segunda dependencia es **Actuator**
2. Luego, en la clase principal de la aplicación que recién acabamos de crear añadiremos la anotación **@EnableConfigServer** 

### Crear repositorio de github

Antes de continuar con la conexión de los microservicios al config server vamos a crear un repositorio en github donde almacenaremos todos los archivos de configuración, para después  **conectar el Config Server al repositorio de github**.
1. Creamos un repositorio de github e introducimos los archivos de configuración:
![](/assets/images/Spring_Cloud_Config/Pasted image 20241216174450.png)
2. En este repositorio he introducido los application_prod.yml y application_qa.yml de cada microservicio. Fíjate en los nombres de los archivos que he cambiado `application` por el nombre del microservicio correspondiente. 

Para este ejemplo he creado solo 2 microservicios(Uno se llama events, otro eventsReviews) y cada uno de estos tiene su configuración para `producción`:
![](/assets/images/Spring_Cloud_Config/Pasted image 20241216180052.png)
Y su configuración para `qa`:
![](/assets/images/Spring_Cloud_Config/Pasted image 20241216180104.png)
El otro microservicio tiene el mismo código para ambos archivos de configuración, como ves es un código muy sencillo simplemente para no irnos por las ramas, pero en cada entorno habrá información distinta como bases de datos, contraseñas ...

Ah, y debo recalcar que este repositorio es público, pero en un entorno real obviamente será privado.

#### Conexión Config Server - Github
Ahora que ya tenemos creado el **config server** y el repositorio de github con los archivos de configuración, el siguiente paso es conectar ambos componentes

1. En el application.yml del config server añadiremos lo siguiente:
```yml
server:  
  port: 8065  
  
spring:  
  application:  
    name: "configserver"  
  profiles:  
    active: git  
  cloud:  
    config:  
      server:  
        git:  
          uri: "https://github.com/egarmar1/events-config.git"  
          default-label: main  
          timeout: 5  
          clone-on-start: true  
          force-pull: true  
  
management:  
  endpoints:  
    web:  
      exposure:  
        include: "*"
```
- Activaremos el perfil de `git` con la propiedad `spring.profiles.active`
- Además, especificaremos el proyecto en el que se encuentran los archivos de configuración, con las propiedades de `spring.cloud.config.server.git`. Concretamente indicaremos la uri, la rama `main`, el tiempo máximo de espera de conexión `5`, si queremos clonar el repositorio al iniciar y así tener los archivos disponibles desde el inicio, y por último indicar que queremos hacer un git pull cada vez que solicitemos las configuraciones
- Por último habilitaremos todos los endpoints de `/actuator` con la propiedad de **management** para poder comprobar que nos conectamos correctamente con el github

##### Comprobar conexión
La conexión entre el config server y github se puede comprobar accediendo a `http://{configServer}/{nombreMicroservicio}/{entorno}`
En nuestro caso sería: `http://localhost:8065/eventsReviews/qa`.
Y si se nos muestra la información del archivo eventsReviews-qa.yml que se encuentra en el gitub, entonces significa que la conexión funciona correctamente


- Tenemos el **config server** conectado con **github**, pero claro, ahora los microservicios deben  conectarse al config server.
#### Conexión Microservicio - Config Server
Tendremos que realizar estos pasos en cada microservicio al que queramos conectar con el config server(Básicamente todos los MS que tengas)
1. Añadir la dependencia de **Config Client**:
```xml
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-config</artifactId>  
</dependency>
```
2. Si no has añadido ninguna funcionalidad de spring cloud en tu microservicio deberás añadir un dependencyManagement justo debajo de las dependencias:
```xml
<dependencyManagement>  
    <dependencies>
           <dependency>
                <groupId>org.springframework.cloud</groupId>  
			    <artifactId>spring-cloud-dependencies</artifactId>  
		        <version>2023.0.3</version>  
		        <type>pom</type>  
				<scope>import</scope>  
		    </dependency>
	</dependencies>
</dependencyManagement>
```

Para saber la última versión de spring cloud puedes crear un nuevo proeycto con esta dependencia en initializr, y al clickar `Explore` verás el dependencyManagement con a última versión.

3. Por último, recuerdas que en el primer apartado hemos especificado los archivos de configuración de esta manera? 
```yml
spring:  
  application:  
    name: "eventReviews"  
  config:  
    import:  
      - "application_prod.yml"  
      - "application_qa.yml"  
  profiles:  
    active:  
      "qa"
```
Bueno, pues ahora en vez de especificar los archivos de configuración directamente, especificaremos la dirección del **Config Server**

```yml
spring:  
  application:  
    name: "eventReviews"  
  config:  
    import:  
      - "optional:configserver:http://localhost:8065/"  
  profiles:  
    active:  
      "qa"
```
He puesto **optional** en el import para que no salte ninguna excepción en el microservicio en el caso de que por cualquier razón el **Config Server** tarde en conectarse.


#### ¿Ya está? ¿Seguro?
Sí , ya está, ya tenemos sincronizado **github** con el **config client**, y este con los **microservicios**.

Aunque aún enfrentamos un par de problemas:
1. ¿Qué ocurre cuando tenemos propiedades con información sensible en los archivos de configuración de github? Es decir, Información sensible que solo queremos que se pueda ver desde la aplicación. Este es el primer problema
2. El segundo problema es que si realizamos algún cambio en el github, entonces tenemos que reiniciar el microservicio completamente para poder ver los cambios realizados.

### Encriptar datos sensibles

Encriptar los datos con spring cloud config es extremadamente sencillo, solo hay que seguir unos pasos:
1. Generar una key: Crea una a través de cualquier herramienta online o inventate tu cualquier key. Por ejemplo: `m89qNuh10NEu8wrfikIL`
2. Introduce la clave en el **application.yml** del spring cloud config server:
```yml
encrypt:  
  key: "m89qNuh10NEu8wrfikIL"
```
3. Ahora para encriptar la información deberás enviarla en el body de un POST que apunte a `http://{configEndpoint}/encrypt`. En mi caso mi config server se encuentra en el 8065, por lo que para encriptar información realizaré los posts a http://localhost:8065/encrypt
![](/assets/images/Spring_Cloud_Config/Pasted image 20241221165549.png)
4. Ahora que ya está encriptado podemos añadirlo de manera segura al github:![](/assets/images/Spring_Cloud_Config/Pasted image 20241221165649.png)
5. Cuando el config server quiera leer este archivo llamará al endpoint `/decrypt`:
![](/assets/images/Spring_Cloud_Config/Pasted image 20241221165803.png)
Y así obtendrá esa información guardada, de hecho, si accedemos a `http://localhost:8065/event/qa` se puede observar la contraseña en texto plano:![](/assets/images/Spring_Cloud_Config/Pasted image 20241221165857.png)
Esto ocurre debido a que por detrás el config server está llamando al endpoint `/decrypt`.

6. Por si te lo estabas preguntando, no, no se puede acceder a este endpoint desde cualquier lugar, en un proyecto real este endpoint estará cerrado y solo se podrá acceder desde dentro del cluster, si no cualquier persona podría desencriptar la información


### Actualizar los archivos de configuración en tiempo de ejecución
El segundo problema que hemos mencionado esque hay que reinciar los microservicios para que obtengan la información cambiada del github, esto es cuanto menos inconveniente.

- Lo que podemos hacer es avisar a los microservicios de un cambio en los archivos de configuración realizando un GET al endpoint `http://{microservice}/actuator/refresh`
- Y se hace con un simple paso, que es, añadir el componente **actuator**(añadiendo su dependencia) en el microservicio, y habilitando sus endpoints:
```yml
management:  
  endpoints:  
    web:  
      exposure:  
        include: "*"
```

- Con actuator añadido, y los endpoints disponibles, al realizar un GET al endpoint `http://{microservicio}/refresh`, este refrescará su información con la que se encuentre en el github.

### Actualizar todos los microservicios ( Spring Cloud Bud)
Ya sabemos como refrescar un microservicio, pero de esta manera es necesario ejecutar el endpoint `/refresh` en cada uno de los microservicios.

Aquí es cuando entra **Spring Cloud Bus**.

- **Spring Cloud Bus** conectará todos los microservicios con un sistema de mensajería (como RabbitMQ, por ejemplo). 

- Cuando se realiza un cambio en los archivos de configuración centralizados (almacenados en GitHub, en este caso) y se notifica a un microservicio mediante una llamada a su endpoint `/actuator/busrefresh`, el message broker distribuye un evento a través del bus para informar al resto de los microservicios conectados sobre el cambio en la configuración. Esto permite que los servicios actualicen su configuración sin necesidad de reiniciarlos manualmente.


#### Implementar Spring Cloud Bus
1. Lo primero que necesitamos es un message broker, he escogido RabbitMQ por su facilidad de poner en marcha, ya que podemos inicializarlo en un contenedor docker ejecutando el siguiente comando:
```sh
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management
```
2. En cada uno de los microservicios especificaremos las configuraciones para conectar el microservicio al message broker:
```yml
spring:  
  rabbitmq:  
    host: "localhost"  
    port: 5672  
    username: "guest"  
    password: "guest"
```
Y recuerda que para consectarse con rabbit es necesario añadir en el pom.xml del microservicio las dependencias de stream: 
```xml
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-stream</artifactId>  
    <version>4.1.3</version>  
</dependency>  
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>  
    <version>4.1.3</version>  
</dependency>
```

3. Por último añadimos la dependencia de `spring cloud starter bus amqp`:
```xml
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-bus-amqp -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    <version>4.2.0</version>
</dependency>

```


- Ahora enviando un post al endpoint `http://{microservice}/actuator/busrefresh` de cualquier microservicio que esté conectado al message broker provocará que este junto con el resto de microservicios se actualizen con el último estado de los archivos de configuración

## Extras
Podríamos seguir con Spring Cloud Config, ya que este tiene más características, como por ejemplo actualizar automáticamente todos los microservicios con el simple hecho de realizar un cambio en el github.

Considero que mostrando más características el tutorial se iría demasiado por las ramas.

Espero que te haya sido útil!!