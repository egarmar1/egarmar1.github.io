---
layout: single
title: Estados de las entidades en JPA
excerpt: "Si estás utilizando JPA como especificación para obtener los datos de la base de datos y convertirlos en POJOs gracias a Hibernate, entonces DEBERÍAS comprender los distintos estados por los que pasa un objeto Entidad."

date: 2024-10-03
classes: wide
header:
  teaser: /assets/images/JPAStates/hibernate.jpg
  teaser_home_page: true
  icon: /assets/images/hibernate.jpg
categories:
  - JPA
  - Hibernate
  - SpringBoot
tags:  
  - SpringBoot
---

## Introducción

Si estás utilizando JPA como especificación para obtener los datos de la base de datos y convertirlos en POJOs gracias a Hibernate, entonces DEBERÍAS comprender los distintos estados por los que pasa un objeto Entidad.


## Estados de las entidades en JPA
Veamos los 4 posibles estados de una entidad en JPA, y ya luego pasaremos a una demo(ejemplo)
### Transient
Este es el estado inicial cuando un objeto es creado. Aquí el objeto está guardado en memoria, pero no está persistido en la base de datos, por lo que si cierras la apliación, este objeto se perderá

### Persistent
En este estado el objeto se sincroniza con la base de datos, y cualquier cambio que hagamos en este, se verá reflejado en la base de datos.

A este estado se puede llegar con uno de estos métodos del `EntityManager`: 
1. El método `persist()` si es un objeto que no existe en la base de datos.
2. Con el método `merge()` si ya existía este objeto en la base de datos.
3. O por otra parte cuando obtenemos un objeto con el método `find()` 

Probablemente estés pensando... *Pero si yo nunca utilizo persist() ni merge() ni find()* y esto es porque estarás utilizando `Spring Data JPA`  con el que solo hay que ejecutar el método `save()`, y por detrás se encarga de  elegir persist() o merge() en cuestión de si ese objeto lleva definido el id o no.
Y por otra parte `findById()` por detrás está llamando al `find()` del EntityManager

```java
@Transactional  
public <S extends T> S save(S entity) {  
    Assert.notNull(entity, "Entity must not be null");  
    if (this.entityInformation.isNew(entity)) {  
        this.entityManager.persist(entity);  
        return entity;  
    } else {  
        return this.entityManager.merge(entity);  
    }  
}
```

Este es el método `save()` que como vemos por detrás escoge `persist()` si es nuevo, o `merge()` si no lo es.

### Detached
Con el estado `Detached`' desconectamos' el objeto de la base de datos, y si se realiza cualquier cambio a este objeto, este cambio no se reflejará en la base de datos. Esto te puede interesar si quieres realizar múltiples cambios con cálculos, ifs ...  antes de volver a guardarlo en la base de datos.

Como hemos visto antes, al hacer `findById()` nuestro objeto está en el estado `Persistent`, para sacarlo de ese estado y moverlo al estado `Detached` utilizaremos el método `detach()` del Bean `EntityManager`.

### Removed
Que un objeto se encuentre en el estado `removed` no significa que el objeto se haya eliminado, si no que el objeto está marcado para ser eliminado.

Cuando realizas un `remove()` con el Bean`EntityManager` o en el caso de usar` Data Spring JPA` usas `delete()` o `deleteById()` (En los cuales se ejecuta remove() por detrás), cuando ejecutas estos métodos, el objeto no se elimina automáticamente, más bien se  marca como candidato a eliminar, en ese mismo momento se comienza una transacción implícita, y al eliminarse ese objeto ya deja de estar en el estado `removed` y pasa a la basura, eso sí, en la memoria de Java sigue estando vigente.


## Demo
La teoría se puede hacer compleja y pesada, pero cuando veas un ejemplo lo vas a entender perfectamente.

Imaginémonos que tenemos una entidad Users, algo así:

```java
@Entity  
@AllArgsConstructor @NoArgsConstructor  
@ToString @Getter @Setter  
public class Users {  
  
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) 
    private String id;  
    private String username;  
    private String password;  
    private String email;  
  
}
```

1. Primero instanciemos un objeto de esta clase:
```java
public void demo(UserRepository userRepository, String userId){ {  
    Users user =new Users("User413", "password123", "user413@email");  
}
```

![](/assets/images/JPAStates/Pasted image 20241016153013.png)
2. Al instanciar la entidad `user` creamos un objeto en memoria, pero jpa por ahora no gestiona ni guarda nada en la base de datos, es decir, la entidad `user` está en el estado `Transient`


```java
public void demo(UserRepository userRepository, String userId){ {  
    Users user =new Users("User413", "password123", "user413@email");  
    userRepository.save(user);  
}
```
Ahora sí, la entiedad `user` ha sido guardada en la base de datos, y se encuentra en el estado `Persistent`
![](/assets/images/JPAStates/Pasted image 20241016153739.png)

Y ahora te hago una pregunta, que ocurrirá si cambio algún atributo del objeto? se verá reflejado en la base de datos?:

```java
public void demo(UserRepository userRepository, String userId){ {  
    Users user =new Users("User413", "password123", "user413@email");  
    userRepository.save(user);  
    userRepository.setUsername("OtherUsername");
}
```
Sí, si que se verá reflejado en la base de datos, ya que está en el estado `Persistent`.

3. Si no queremos que cada cambio se manifieste en la base de datos entonces necesitamos que la entidad `user` pase a `Detached`, cómo lo hacemos? así:

```java
public void demo(UserRepository userRepository, String userId, EntityManager entityManager){ {  
    Users user =new Users("User413", "password123", "user413@email");  
    userRepository.save(user);

    entityManager.detach(user);
    userRepository.setUsername("OtherUsername");
}
```
Es decir, inyectando el entityManager, y usando el método `detach()`, de esta manera el setUsername no cambiará el nombre del usuario en la base de datos, necesitaremos hacer otro `save()` para que la base de datos registre este cambio

![](/assets/images/JPAStates/Pasted image 20241016153828.png)

4. Por último, si quieres que se elimine el objeto, entonces este deberá de pasar antes por el estado `Removed`:
```java
public void demo(UserRepository userRepository, String userId, EntityManager entityManager){ {  
    Users user =new Users("User413", "password123", "user413@email");  
    userRepository.save(user);
    userRepository.setUsername("OtherUsername");

	userRepository.deleteById(user.getId());
}
```

![](/assets/images/JPAStates/Pasted image 20241016153910.png)

Una vez se elimine de la base de datos, JPA se desentiende de esta entidad(es decir que ya no está en ningún estado), pero seguimos teniendo el objeto en memoria, es decir en la aplicación de Java.



## Conclusión
Con este tutorial has podido comprender al máximo detalle cómo funcionan los estados de las entidades de JPA, concepto super importante, que OBLIGATORIAMENTE tienes que saber si quieres trabajar con Spring Boot, y concretamente con JPA, Hibernate y Data Spring JPA.

Muchas gracias por tu tiempo, y si te ha gustado, tienes muchos más posts en esta propia web :)