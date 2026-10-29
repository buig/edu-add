# UP6: Creación de una API REST con Spring Boot

## 1. Punto de partida

En las unidades anteriores ya hemos recorrido todo lo necesario para construir una API REST completa, aunque lo hayamos visto en piezas sueltas:

- En la **UP3** aprendimos qué es Spring Boot, el patrón **MVC**, cómo crear un controlador y cómo recibir parámetros de distintas formas (consulta, ruta y cuerpo de la petición).
- En la **UP4** aprendimos a mapear **entidades** con Hibernate/JPA y a gestionar su ciclo de vida con el `EntityManager`.
- En la **UP5** aprendimos a modelar **relaciones** entre entidades (1 a 1, 1 a N y N a M) y a controlar cómo se cargan esos datos (*lazy* / *eager*).

En esta unidad vamos a **unir todas estas piezas** para construir, paso a paso, una **API REST completa** sobre nuestro modelo de datos `Autor` / `Libro` (el mismo que empezamos a trabajar en la UP5), siguiendo las prácticas habituales en la industria.

!!! note "¿Qué NO vamos a usar?"
    No vamos a utilizar **HATEOAS** (*Hypermedia as the Engine of Application State*). HATEOAS es una extensión de REST en la que cada respuesta incluye enlaces a las acciones disponibles sobre ese recurso (por ejemplo, el propio recurso incluiría un enlace para editarse o eliminarse a sí mismo).

    Es un concepto interesante, pero añade una complejidad que no aporta nada a nuestro objetivo de esta unidad: aprender a construir y consumir una API REST "plana", que es la que te vas a encontrar en la gran mayoría de proyectos profesionales.

---

## 2. ¿Qué es una API REST?

**REST** (*Representational State Transfer*) es un estilo de arquitectura para diseñar servicios web. No es un protocolo ni una tecnología concreta, sino un conjunto de **convenciones** que, si las seguimos, hacen que nuestra API sea predecible, coherente y fácil de usar por cualquier cliente (una app móvil, un frontend web, otro servicio, Postman...).

Las ideas clave de REST son:

- **Recursos**: todo lo que la API expone se modela como un **recurso** (un `Libro`, un `Autor`, una `Reserva`...), identificado por una URL. Por ejemplo: `/api/libros/7` identifica al libro con `id = 7`.
- **Verbos HTTP**: la **acción** que queremos realizar sobre el recurso no va en la URL, sino que se expresa con el **método HTTP** de la petición.
- **Sin estado (*stateless*)**: cada petición debe contener toda la información necesaria para ser procesada. El servidor no "recuerda" nada entre peticiones (si hace falta identificar al usuario, se hace en cada petición, normalmente mediante un token).
- **Representación**: el recurso se envía y recibe en un formato estándar, normalmente **JSON**.

### 2.1. Verbos HTTP y operaciones CRUD

| Verbo HTTP | Operación CRUD | Ejemplo de URL | Descripción |
|---|---|---|---|
| `GET` | Leer (*Read*) | `GET /api/libros` | Devuelve la lista de libros |
| `GET` | Leer (*Read*) | `GET /api/libros/7` | Devuelve el libro con id 7 |
| `POST` | Crear (*Create*) | `POST /api/libros` | Crea un nuevo libro |
| `PUT` | Actualizar (*Update*) | `PUT /api/libros/7` | Sustituye por completo el libro 7 |
| `PATCH` | Actualizar parcial | `PATCH /api/libros/7` | Modifica solo algunos campos del libro 7 |
| `DELETE` | Eliminar (*Delete*) | `DELETE /api/libros/7` | Elimina el libro 7 |

!!! tip "Convenciones de nombrado de rutas"
    - Los recursos se nombran en **plural**: `/api/libros`, no `/api/libro`.
    - La **acción nunca va en la URL**. Es incorrecto escribir `/api/crearLibro` o `/api/libros/eliminar/7`; la acción la determina el verbo HTTP, no la ruta.
    - Los recursos anidados reflejan una relación de pertenencia: `/api/autores/3/libros` para "los libros del autor 3".
    - Las URLs se escriben en minúsculas y, si hace falta separar palabras, con guiones (`-`), no con guion bajo ni camelCase.

### 2.2. Idempotencia

Un detalle importante: `GET`, `PUT` y `DELETE` deben ser **idempotentes**: repetir la misma petición varias veces produce siempre el mismo resultado (el recurso queda en el mismo estado). `POST` **no** es idempotente: cada llamada crea un recurso nuevo. Esto no es un detalle anecdótico: es lo que permite, por ejemplo, que un cliente pueda reintentar una petición `GET` o `DELETE` sin miedo a duplicar datos.

---

## 3. Arquitectura en capas

Hasta ahora, en la UP4 y la UP5, trabajábamos con el `EntityManager` directamente desde una clase de prueba (`CommandLineRunner`). Eso está bien para aprender cómo funciona Hibernate por dentro, pero **no es así como se construye una aplicación real**.

Una API REST con Spring Boot se organiza en **capas**, cada una con una única responsabilidad:

```
Cliente (Postman, app, frontend...)
        │  petición HTTP
        ▼
   CONTROLLER   → traduce HTTP ↔ Java. No contiene lógica de negocio.
        │
        ▼
    SERVICE      → contiene la lógica de negocio. No sabe nada de HTTP.
        │
        ▼
   REPOSITORY    → accede a los datos (lo que antes hacíamos con EntityManager).
        │
        ▼
   BASE DE DATOS
```

!!! note "¿Por qué separar en capas?"
    - **Mantenibilidad**: si cambia la forma de exponer la API (por ejemplo, pasamos de REST a gRPC), solo tocamos el `Controller`; el `Service` no cambia.
    - **Reutilización**: la misma lógica de negocio del `Service` podría usarse desde un `Controller` REST y, a la vez, desde una tarea programada o un `CommandLineRunner`, sin duplicar código.
    - **Testing**: podemos probar el `Service` con tests unitarios sin necesidad de levantar un servidor HTTP.

---

## 4. La capa Repository: Spring Data JPA

En la UP4 trabajamos directamente con `EntityManager`. Spring Data JPA nos ofrece una abstracción mucho más cómoda: basta con **declarar una interfaz**, y Spring genera la implementación por nosotros en tiempo de ejecución.

```java
package com.iessalvadorgadea.biblioteca.repositories;

import com.iessalvadorgadea.biblioteca.entities.Libro;
import org.springframework.data.jpa.repository.JpaRepository;

public interface LibroRepository extends JpaRepository<Libro, Integer> {
}
```

Con solo esto, ya disponemos de métodos como:

| Método heredado | Qué hace |
|---|---|
| `save(libro)` | Inserta o actualiza (según tenga o no `id`) |
| `findById(id)` | Devuelve un `Optional<Libro>` |
| `findAll()` | Devuelve `List<Libro>` con todos los registros |
| `deleteById(id)` | Elimina por id |
| `existsById(id)` | Comprueba si existe un registro con ese id |
| `count()` | Cuenta el número total de registros |

`JpaRepository<Libro, Integer>` indica: *"gestiono la entidad `Libro`, cuya clave primaria es de tipo `Integer`"*.

### 4.1. *Query methods*: consultas derivadas del nombre del método

Spring Data JPA puede generar la consulta SQL a partir del **nombre del método**, siguiendo unas convenciones:

```java
public interface LibroRepository extends JpaRepository<Libro, Integer> {

    List<Libro> findByTitulo(String titulo);

    List<Libro> findByAutor(Autor autor);

    List<Libro> findByAutor_Id(Integer autorId);

    List<Libro> findByTituloContainingIgnoreCase(String fragmento);

    boolean existsByTitulo(String titulo);
}
```

!!! tip "¿Cómo lo hace Spring sin escribir ni una línea de SQL?"
    Spring Data JPA analiza el nombre del método (`findBy`, `existsBy`...) y las propiedades de la entidad que le siguen (`Titulo`, `Autor_Id`...) y construye la consulta JPQL equivalente automáticamente. Es exactamente el mismo mecanismo que usa `@Entity` y `@Column` para traducir atributos Java a columnas SQL.

---

## 5. La capa Service

El `Service` es donde vive la **lógica de negocio**: validaciones que no son puramente de formato, reglas, y la coordinación entre varios repositorios si fuera necesario. Además, es el lugar natural para gestionar las **transacciones**.

```java
package com.iessalvadorgadea.biblioteca.services;

import com.iessalvadorgadea.biblioteca.entities.Libro;
import com.iessalvadorgadea.biblioteca.repositories.LibroRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class LibroService {

    private final LibroRepository libroRepository;

    public LibroService(LibroRepository libroRepository) {
        this.libroRepository = libroRepository;
    }

    @Transactional(readOnly = true)
    public List<Libro> listarTodos() {
        return libroRepository.findAll();
    }

    @Transactional(readOnly = true)
    public Libro buscarPorId(Integer id) {
        return libroRepository.findById(id)
                .orElseThrow(() -> new RecursoNoEncontradoException("Libro no encontrado con id " + id));
    }

    @Transactional
    public Libro crear(Libro libro) {
        return libroRepository.save(libro);
    }

    @Transactional
    public Libro actualizar(Integer id, Libro datosNuevos) {
        Libro libroExistente = buscarPorId(id);
        libroExistente.setTitulo(datosNuevos.getTitulo());
        libroExistente.setAnioPublicacion(datosNuevos.getAnioPublicacion());
        libroExistente.setAutor(datosNuevos.getAutor());
        return libroRepository.save(libroExistente);
    }

    @Transactional
    public void eliminar(Integer id) {
        if (!libroRepository.existsById(id)) {
            throw new RecursoNoEncontradoException("Libro no encontrado con id " + id);
        }
        libroRepository.deleteById(id);
    }
}
```

!!! note "Inyección de dependencias por constructor"
    Fíjate en que `LibroRepository` se recibe en el **constructor**, no se anota el atributo con `@Autowired`. Es la forma recomendada de inyectar dependencias en Spring: hace que la clase sea más fácil de testear (se puede construir pasando un *mock* del repositorio) y deja claro, solo con leer el constructor, de qué depende la clase.

### 5.1. `@Transactional`: por qué es importante aquí

Ya vimos en la UP5 el problema de la carga `Lazy`: si intentamos acceder a una colección `Lazy` (por ejemplo, `autor.getLibros()`) **fuera** de una sesión de Hibernate activa, obtenemos un `LazyInitializationException`. Anotar los métodos del `Service` con `@Transactional` asegura que, mientras dura el método, la sesión de Hibernate está abierta, y por tanto es un buen lugar para acceder a colecciones relacionadas.

---

## 6. DTOs: por qué no devolver las entidades directamente

Podría parecer que lo más directo es que el `Controller` devuelva directamente nuestras entidades (`Libro`, `Autor`) convertidas a JSON. **No es una buena práctica**, y en nuestro caso concreto, con relaciones bidireccionales, puede llegar a **romper la aplicación**. Las razones:

1. **Recursión infinita**: si `Libro` tiene un `Autor`, y `Autor` tiene una lista de `Libro`, al serializar un `Libro` a JSON, Jackson serializa su `Autor`, que a su vez serializa sus `Libro`, que a su vez serializan su `Autor`... hasta un `StackOverflowError`. Ya avisamos de este problema con las relaciones bidireccionales en la UP5, y aquí es donde nos pasa factura si no lo resolvemos.
2. **`LazyInitializationException`**: si alguna colección `Lazy` no se ha cargado y la sesión de Hibernate ya se ha cerrado cuando Jackson intenta serializarla (algo muy habitual, porque la serialización ocurre **después** de que el método del `Service` haya terminado), la petición falla con un error 500.
3. **Exponemos más de lo necesario**: la entidad puede tener campos internos (contraseñas, campos de auditoría, claves foráneas en bruto) que no queremos que viajen por la API.
4. **Acoplamos la API a la base de datos**: si cambiamos un nombre de columna o una relación en la entidad, cambiamos automáticamente el contrato de nuestra API, sin quererlo.

### 6.1. ¿Qué es un DTO?

Un **DTO** (*Data Transfer Object*) es una clase sencilla, sin lógica, cuyo único propósito es **transportar datos** entre capas (en nuestro caso, entre el `Service`/`Controller` y el cliente de la API). Con Java moderno, el candidato ideal para un DTO es un **record**: inmutable, conciso, y con `equals`/`hashCode`/`toString` generados automáticamente.

```java
package com.iessalvadorgadea.biblioteca.dto;

public record LibroDTO(
        Integer id,
        String titulo,
        Integer anioPublicacion,
        Integer autorId,
        String nombreAutor
) {
}
```

Fíjate en el detalle clave: en lugar de incluir el objeto `Autor` completo (que nos llevaría otra vez al problema de la recursión), incluimos únicamente `autorId` y `nombreAutor`: los datos **planos** que un cliente de la API necesita, sin arrastrar la relación completa.

Para crear el DTO necesitamos otro, para el que se recibe al crear/actualizar un libro (no necesita `id`, lo genera la base de datos):

```java
public record LibroEntradaDTO(
        String titulo,
        Integer anioPublicacion,
        Integer autorId
) {
}
```

### 6.2. El mapeo Entidad ↔ DTO

El mapeo lo podemos hacer a mano (es perfectamente razonable para un proyecto de este tamaño) con un método estático o de instancia:

```java
public class LibroMapper {

    public static LibroDTO aDTO(Libro libro) {
        return new LibroDTO(
                libro.getId(),
                libro.getTitulo(),
                libro.getAnioPublicacion(),
                libro.getAutor().getId(),
                libro.getAutor().getNombre()
        );
    }
}
```

!!! note "¿Y si el proyecto crece mucho?"
    En proyectos grandes se suelen usar librerías como **MapStruct** para generar este mapeo automáticamente. No es necesario para esta unidad, pero es bueno que sepas que existe si en el futuro trabajas en un proyecto con muchas entidades y DTOs.

---

## 7. El Controller REST completo

Con el `Service` y los DTOs ya preparados, el `Controller` queda muy limpio: su única responsabilidad es traducir entre HTTP y Java.

```java
package com.iessalvadorgadea.biblioteca.controllers;

import com.iessalvadorgadea.biblioteca.dto.LibroDTO;
import com.iessalvadorgadea.biblioteca.dto.LibroEntradaDTO;
import com.iessalvadorgadea.biblioteca.dto.LibroMapper;
import com.iessalvadorgadea.biblioteca.entities.Libro;
import com.iessalvadorgadea.biblioteca.services.LibroService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/libros")
public class LibroController {

    private final LibroService libroService;

    public LibroController(LibroService libroService) {
        this.libroService = libroService;
    }

    @GetMapping
    public List<LibroDTO> listarTodos() {
        return libroService.listarTodos()
                .stream()
                .map(LibroMapper::aDTO)
                .toList();
    }

    @GetMapping("/{id}")
    public ResponseEntity<LibroDTO> buscarPorId(@PathVariable Integer id) {
        Libro libro = libroService.buscarPorId(id);
        return ResponseEntity.ok(LibroMapper.aDTO(libro));
    }

    @PostMapping
    public ResponseEntity<LibroDTO> crear(@Valid @RequestBody LibroEntradaDTO entrada) {
        Libro creado = libroService.crear(entrada);
        return ResponseEntity.status(HttpStatus.CREATED).body(LibroMapper.aDTO(creado));
    }

    @PutMapping("/{id}")
    public ResponseEntity<LibroDTO> actualizar(@PathVariable Integer id,
                                                @Valid @RequestBody LibroEntradaDTO entrada) {
        Libro actualizado = libroService.actualizar(id, entrada);
        return ResponseEntity.ok(LibroMapper.aDTO(actualizado));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> eliminar(@PathVariable Integer id) {
        libroService.eliminar(id);
        return ResponseEntity.noContent().build();
    }
}
```

!!! tip "`@RestController` vs `@Controller` + `@ResponseBody`"
    Recuerda de la UP3 que `@RestController` es exactamente `@Controller` + `@ResponseBody` aplicado a todos los métodos de la clase. Como aquí **toda** la clase es una API, usamos siempre `@RestController`.

---

## 8. Códigos de estado HTTP y `ResponseEntity`

Una API REST bien diseñada **no devuelve siempre `200 OK`**. El código de estado comunica al cliente, sin que tenga que leer el cuerpo de la respuesta, qué ha pasado.

| Código | Nombre | Cuándo se usa |
|---|---|---|
| `200 OK` | Correcto | `GET` o `PUT` que terminan bien |
| `201 Created` | Creado | `POST` que crea un recurso nuevo |
| `204 No Content` | Sin contenido | `DELETE` correcto (no hay nada que devolver) |
| `400 Bad Request` | Petición incorrecta | Los datos recibidos no son válidos |
| `404 Not Found` | No encontrado | Se pide un recurso que no existe |
| `409 Conflict` | Conflicto | El recurso ya existe o viola una restricción (p.ej. un título duplicado) |
| `500 Internal Server Error` | Error del servidor | Un error inesperado no controlado |

`ResponseEntity<T>` nos permite construir la respuesta completa (código de estado + cabeceras + cuerpo) de forma explícita, en lugar de dejar que Spring decida por nosotros. Compara:

```java
// Sin control del código de estado: Spring siempre responde 200
@GetMapping("/{id}")
public LibroDTO buscarPorId(@PathVariable Integer id) { ... }

// Con control explícito del código de estado
@GetMapping("/{id}")
public ResponseEntity<LibroDTO> buscarPorId(@PathVariable Integer id) { ... }
```

---

## 9. Validación de los datos de entrada

No podemos confiar en que el cliente de la API siempre envíe datos correctos. Spring Boot integra **Bean Validation** (la especificación `jakarta.validation`) para declarar las reglas directamente sobre el DTO de entrada.

Primero, añadimos la dependencia (si no la tenemos ya) en el `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Y anotamos el DTO de entrada:

```java
public record LibroEntradaDTO(

        @NotBlank(message = "El título no puede estar vacío")
        @Size(max = 150, message = "El título no puede superar los 150 caracteres")
        String titulo,

        @NotNull(message = "El año de publicación es obligatorio")
        @Positive(message = "El año de publicación debe ser positivo")
        Integer anioPublicacion,

        @NotNull(message = "El id del autor es obligatorio")
        Integer autorId
) {
}
```

| Anotación | Comprueba |
|---|---|
| `@NotNull` | El valor no es `null` |
| `@NotBlank` | No es `null`, ni vacío, ni solo espacios (para `String`) |
| `@Size(min=, max=)` | Longitud de una cadena o colección |
| `@Positive` / `@Negative` | El número es positivo/negativo |
| `@Email` | Tiene formato de email válido |
| `@Min(valor)` / `@Max(valor)` | Valor numérico mínimo/máximo |

En el `Controller`, basta con añadir `@Valid` delante de `@RequestBody`:

```java
@PostMapping
public ResponseEntity<LibroDTO> crear(@Valid @RequestBody LibroEntradaDTO entrada) { ... }
```

Si la validación falla, Spring lanza automáticamente una `MethodArgumentNotValidException` **antes** de que el código del método se ejecute, y responde con un `400 Bad Request`. En el siguiente apartado veremos cómo personalizar esa respuesta de error.

---

## 10. Manejo global de errores

Sin hacer nada más, cuando algo falla (una validación, una excepción nuestra, un error inesperado), Spring Boot devuelve una página de error genérica (la típica *Whitelabel Error Page* que ya vimos en la UP3) o un JSON poco informativo. Vamos a centralizar y mejorar esto con `@RestControllerAdvice`.

### 10.1. Una excepción propia

```java
package com.iessalvadorgadea.biblioteca.exceptions;

public class RecursoNoEncontradoException extends RuntimeException {
    public RecursoNoEncontradoException(String mensaje) {
        super(mensaje);
    }
}
```

### 10.2. Un DTO para representar el error

```java
public record ErrorDTO(
        int codigo,
        String mensaje,
        java.time.LocalDateTime fecha
) {
}
```

### 10.3. El manejador global

```java
package com.iessalvadorgadea.biblioteca.exceptions;

import com.iessalvadorgadea.biblioteca.dto.ErrorDTO;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.stream.Collectors;

@RestControllerAdvice
public class ManejadorGlobalDeErrores {

    @ExceptionHandler(RecursoNoEncontradoException.class)
    public ResponseEntity<ErrorDTO> manejarNoEncontrado(RecursoNoEncontradoException ex) {
        ErrorDTO error = new ErrorDTO(HttpStatus.NOT_FOUND.value(), ex.getMessage(), LocalDateTime.now());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorDTO> manejarValidacion(MethodArgumentNotValidException ex) {
        String mensaje = ex.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining(" | "));
        ErrorDTO error = new ErrorDTO(HttpStatus.BAD_REQUEST.value(), mensaje, LocalDateTime.now());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorDTO> manejarGenerico(Exception ex) {
        ErrorDTO error = new ErrorDTO(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                "Ha ocurrido un error inesperado",
                LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

!!! warning "Importante: orden y especificidad"
    Spring elige el `@ExceptionHandler` más específico para la excepción lanzada. Por eso podemos tener un manejador genérico para `Exception` como red de seguridad, y manejadores concretos para los casos que queremos tratar de forma particular (`RecursoNoEncontradoException`, `MethodArgumentNotValidException`...). El manejador de `Exception` nunca "roba" los casos específicos: Spring siempre prioriza la coincidencia más concreta.

Con este `@RestControllerAdvice`, **todos** los controladores de la aplicación (no solo `LibroController`) comparten el mismo manejo de errores, consistente y centralizado.

---

## 11. Relaciones en la API: el caso Autor-Libro

Recuperamos el modelo de la UP5: un `Autor` tiene muchos `Libro` (1 a N). ¿Cómo lo exponemos en la API sin caer en los problemas del apartado 6?

### 11.1. Endpoint anidado

Es una convención muy habitual exponer los libros de un autor como un recurso **anidado** bajo el autor:

```java
@RestController
@RequestMapping("/api/autores")
public class AutorController {

    private final AutorService autorService;

    public AutorController(AutorService autorService) {
        this.autorService = autorService;
    }

    // ... CRUD de Autor (igual que el de Libro) ...

    @GetMapping("/{id}/libros")
    public List<LibroDTO> librosDelAutor(@PathVariable Integer id) {
        return autorService.buscarPorId(id)
                .getLibros()
                .stream()
                .map(LibroMapper::aDTO)
                .toList();
    }
}
```

Como este método está dentro de un método del `Service` marcado con `@Transactional`, podemos acceder sin problema a `getLibros()` aunque la colección sea `Lazy`, porque la sesión de Hibernate sigue abierta.

### 11.2. El DTO de Autor no incluye la lista completa de libros

Si `AutorDTO` incluyera una lista de `LibroDTO`, y cada `LibroDTO` tuviera el `autorId`, no habría recursión (porque son DTOs "planos", no las entidades), pero normalmente **tampoco es lo que queremos**: cuando alguien pide `GET /api/autores/3`, probablemente no quiera descargar también los 200 libros de ese autor. Por eso se prefiere:

```java
public record AutorDTO(
        Integer id,
        String nombre,
        String nacionalidad
) {
}
```

Y quien quiera los libros de ese autor, los pide explícitamente con `GET /api/autores/3/libros`. Esta es, precisamente, una de las decisiones de diseño que **HATEOAS** intenta resolver añadiendo enlaces automáticamente a cada respuesta — y es la complejidad que hemos decidido no abordar en esta unidad.

---

## 12. Probando la API

Una API no sirve de mucho si no podemos comprobar que funciona. Las dos formas más habituales son:

- Un cliente HTTP externo como **Postman** o **Insomnia**.
- El **cliente HTTP integrado en IntelliJ IDEA** (ficheros `.http`), que es el que vamos a usar en la guía práctica de la siguiente sección, porque no requiere salir del IDE.

---

## 13. Resumen de la unidad

```
 Petición HTTP
      │
      ▼
 ┌─────────────┐     @Valid      ┌─────────────┐
 │ Controller  │ ───────────────▶│   Service   │
 │ (HTTP ↔ DTO)│                 │  (negocio,  │
 └─────────────┘                 │ @Transactional)
      ▲                          └─────────────┘
      │ DTO                             │ Entidad
      │                                 ▼
      │                         ┌─────────────┐
      │                         │ Repository  │
      │                         │(JpaRepository)
      │                         └─────────────┘
      │                                 │
      └──────────── JSON ───────────────┘
```

- **Controller**: traduce HTTP ↔ Java, delega en el `Service`, nunca contiene lógica de negocio.
- **Service**: contiene la lógica de negocio, gestiona las transacciones (`@Transactional`), trabaja con **entidades**.
- **Repository**: accede a los datos mediante `JpaRepository`, sin que tengamos que escribir SQL ni gestionar el `EntityManager` a mano.
- **DTO**: lo que realmente viaja por la red; nunca exponemos las entidades directamente.
- **`@RestControllerAdvice`**: centraliza el manejo de errores de toda la aplicación.

En la siguiente sección vamos a construir este proyecto paso a paso con **IntelliJ IDEA**.
