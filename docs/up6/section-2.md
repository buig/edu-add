# UP6: Guía práctica — Creación del proyecto con IntelliJ IDEA

En esta sección vamos a crear, paso a paso, el proyecto completo de la API REST `Autor`/`Libro` con **IntelliJ IDEA**, el mismo IDE que ya usamos en la UP3, UP4 y UP5.

> 📸 **Nota:** esta guía está pensada para que la sigas creando tu propio proyecto y vayas insertando tus propias capturas de pantalla en cada paso marcado con el icono 📸, sustituyendo este aviso por la imagen correspondiente (`![Descripción](img/capN.png)`), igual que se ha hecho en las unidades anteriores.

---

## 1. Creación del proyecto con Spring Initializr

1. Abre IntelliJ IDEA y selecciona **File → New → Project...**
2. Elige **Spring Boot** en el panel izquierdo (IntelliJ integra el propio Spring Initializr).
3. Rellena los datos del proyecto:
    - **Name**: `biblioteca-api`
    - **Language**: `Java`
    - **Type**: `Maven`
    - **Group**: `com.iessalvadorgadea` (o tu propio dominio invertido, como ya vimos en la UP3)
    - **Artifact**: `biblioteca-api`
    - **Java**: la versión que tengáis instalada en el aula (21 o superior)

> 📸 **Captura sugerida:** la pantalla de configuración inicial del proyecto en IntelliJ.

4. En la pantalla de dependencias, añade:
    - **Spring Web**
    - **Spring Data JPA**
    - **MySQL Driver**
    - **Validation**
    - **Lombok**

> 📸 **Captura sugerida:** la pantalla de selección de dependencias, con las cinco marcadas.

5. Finaliza el asistente. IntelliJ generará el proyecto y descargará las dependencias con Maven.

!!! tip "¿Te suena esta pantalla?"
    Es exactamente la misma pantalla de creación de proyecto que usamos en la UP3, solo que ahora añadimos **Spring Data JPA**, **MySQL Driver** y **Validation**, que no necesitábamos entonces.

---

## 2. Estructura de paquetes del proyecto

Dentro del paquete principal (`com.iessalvadorgadea.bibliotecaapi`), crea la siguiente estructura de subpaquetes, haciendo clic derecho → **New → Package**:

```
com.iessalvadorgadea.bibliotecaapi
├── entities       (las clases @Entity: Autor, Libro)
├── repositories   (las interfaces JpaRepository)
├── services       (la lógica de negocio)
├── controllers    (los @RestController)
├── dto            (los records de entrada/salida)
└── exceptions     (excepciones propias y el manejador global)
```

> 📸 **Captura sugerida:** el árbol de paquetes ya creado en el panel de proyecto de IntelliJ.

---

## 3. Configuración de `application.properties`

Abre `src/main/resources/application.properties` y configura la conexión a la base de datos, igual que hicimos en la UP4:

```properties
# Puerto de escucha
server.port=8080

# Conexión al SGBD
spring.datasource.url=jdbc:mysql://localhost:3308/biblioteca_api?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=toor

# Hibernate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.hibernate.ddl-auto=create
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
```

!!! warning "Ajusta el puerto de MySQL"
    Si tu contenedor Docker de MySQL (UP4) usa un puerto distinto a `3308`, cámbialo aquí. Recuerda que puedes comprobar el puerto expuesto con `docker ps`.

---

## 4. Las entidades: `Autor` y `Libro`

Si ya las tienes creadas de la UP5, puedes copiarlas directamente. Si no, aquí tienes la versión que usaremos como referencia en esta unidad.

### 4.1. `Autor`

```java
package com.iessalvadorgadea.bibliotecaapi.entities;

import jakarta.persistence.*;
import lombok.*;

import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "autores")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Autor {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(nullable = false, length = 100)
    private String nombre;

    @Column(length = 100)
    private String nacionalidad;

    @OneToMany(mappedBy = "autor", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @ToString.Exclude
    @EqualsAndHashCode.Exclude
    private List<Libro> libros = new ArrayList<>();
}
```

### 4.2. `Libro`

```java
package com.iessalvadorgadea.bibliotecaapi.entities;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "libros")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Libro {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(nullable = false, length = 150)
    private String titulo;

    @Column(name = "anio_publicacion")
    private Integer anioPublicacion;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "autor_id", nullable = false)
    @ToString.Exclude
    private Autor autor;
}
```

!!! note "`@ToString.Exclude` y `@EqualsAndHashCode.Exclude`"
    Al igual que avisamos en la UP5 sobre `toString()`/`equals()`/`hashCode()` con relaciones bidireccionales, excluimos el lado `List<Libro>` para que Lombok no genere una recursión infinita al construir el `toString()` de `Autor`.

> 📸 **Captura sugerida:** las dos clases de entidad ya creadas, con el árbol de paquetes a la izquierda.

---

## 5. Los repositorios

```java
package com.iessalvadorgadea.bibliotecaapi.repositories;

import com.iessalvadorgadea.bibliotecaapi.entities.Autor;
import org.springframework.data.jpa.repository.JpaRepository;

public interface AutorRepository extends JpaRepository<Autor, Integer> {
}
```

```java
package com.iessalvadorgadea.bibliotecaapi.repositories;

import com.iessalvadorgadea.bibliotecaapi.entities.Libro;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface LibroRepository extends JpaRepository<Libro, Integer> {
    List<Libro> findByAutor_Id(Integer autorId);
}
```

---

## 6. Los DTOs

Crea en el paquete `dto` los siguientes records (puedes copiarlos literalmente de la sección de teoría):

- `AutorDTO`
- `LibroDTO`
- `LibroEntradaDTO`
- `ErrorDTO`

> 📸 **Captura sugerida:** el paquete `dto` con los cuatro ficheros creados.

---

## 7. Los servicios

Crea `AutorService` y `LibroService` en el paquete `services`, siguiendo el ejemplo de la sección de teoría (apartado 5). No olvides:

- Inyectar el repositorio por **constructor**.
- Anotar la clase con `@Service`.
- Anotar los métodos de lectura con `@Transactional(readOnly = true)` y los de escritura con `@Transactional`.

---

## 8. Los controladores

Crea `AutorController` y `LibroController` en el paquete `controllers`, con las rutas base `/api/autores` y `/api/libros` respectivamente, siguiendo el CRUD completo del apartado 7 de la teoría.

> 📸 **Captura sugerida:** el código completo de `LibroController` en el editor.

---

## 9. El manejador global de errores

Crea la excepción `RecursoNoEncontradoException` y el `ManejadorGlobalDeErrores` en el paquete `exceptions`, copiando el código del apartado 10 de la teoría.

---

## 10. Datos de prueba (`import.sql`)

Como ya hicimos en la UP4, crea `src/main/resources/import.sql` para tener datos desde el arranque:

```sql
INSERT INTO autores (nombre, nacionalidad) VALUES ('Gabriel García Márquez', 'Colombiana');
INSERT INTO autores (nombre, nacionalidad) VALUES ('Isabel Allende', 'Chilena');

INSERT INTO libros (titulo, anio_publicacion, autor_id) VALUES ('Cien años de soledad', 1967, 1);
INSERT INTO libros (titulo, anio_publicacion, autor_id) VALUES ('El amor en los tiempos del cólera', 1985, 1);
INSERT INTO libros (titulo, anio_publicacion, autor_id) VALUES ('La casa de los espíritus', 1982, 2);
```

---

## 11. Ejecutar la aplicación

Ejecuta la clase principal (`BibliotecaApiApplication`) igual que en la UP3. Comprueba en la consola que:

- Hibernate crea las tablas `autores` y `libros`.
- Se insertan los datos de `import.sql`.
- El servidor Tomcat queda escuchando en el puerto `8080`.

> 📸 **Captura sugerida:** la consola de IntelliJ mostrando el arranque correcto, con la creación de tablas y la importación de datos.

---

## 12. Probar la API con el cliente HTTP de IntelliJ

IntelliJ IDEA Ultimate (la versión que usamos en el centro) incluye un cliente HTTP integrado: no necesitas instalar Postman.

1. Crea un nuevo fichero en la raíz del proyecto llamado `peticiones.http`.
2. IntelliJ detecta la extensión `.http` y muestra un icono ▶️ junto a cada petición para ejecutarla directamente.

```http
### Listar todos los libros
GET http://localhost:8080/api/libros

### Buscar un libro por id
GET http://localhost:8080/api/libros/1

### Crear un libro nuevo
POST http://localhost:8080/api/libros
Content-Type: application/json

{
  "titulo": "Rayuela",
  "anioPublicacion": 1963,
  "autorId": 1
}

### Actualizar un libro
PUT http://localhost:8080/api/libros/1
Content-Type: application/json

{
  "titulo": "Cien años de soledad (edición revisada)",
  "anioPublicacion": 1967,
  "autorId": 1
}

### Eliminar un libro
DELETE http://localhost:8080/api/libros/4

### Probar la validación (debe fallar con 400)
POST http://localhost:8080/api/libros
Content-Type: application/json

{
  "titulo": "",
  "anioPublicacion": -5
}

### Listar los libros de un autor
GET http://localhost:8080/api/autores/1/libros

### Buscar un libro que no existe (debe devolver 404 con el ErrorDTO)
GET http://localhost:8080/api/libros/999
```

> 📸 **Captura sugerida:** el fichero `peticiones.http` con alguna petición ejecutada y la respuesta mostrada por IntelliJ en el panel derecho (código de estado, cabeceras y cuerpo JSON).

!!! tip "Guarda las respuestas de ejemplo"
    Para cada operación (`GET`, `POST`, `PUT`, `DELETE` y los dos casos de error), haz una captura mostrando tanto la petición como la respuesta completa (código de estado incluido). Son las capturas que necesitarás para completar esta unidad y, más adelante, para las prácticas.

---

## 13. Checklist final de la unidad

- [ ] El proyecto arranca sin errores y Hibernate crea las tablas.
- [ ] `GET /api/libros` devuelve la lista de libros en formato `LibroDTO` (sin recursión, sin exponer la entidad completa).
- [ ] `GET /api/libros/{id}` devuelve `404` con un `ErrorDTO` cuando el libro no existe.
- [ ] `POST /api/libros` crea un libro y devuelve `201 Created`.
- [ ] `POST /api/libros` con datos inválidos devuelve `400` con el mensaje de validación.
- [ ] `PUT /api/libros/{id}` actualiza el libro y devuelve `200 OK`.
- [ ] `DELETE /api/libros/{id}` elimina el libro y devuelve `204 No Content`.
- [ ] `GET /api/autores/{id}/libros` devuelve los libros de ese autor.

Cuando tengas todo esto funcionando y documentado con tus capturas, la unidad estará lista. A partir de este proyecto base, en las **Prácticas** adaptaremos los enunciados que ya teníais en Symfony al formato Spring Boot que hemos construido aquí.
