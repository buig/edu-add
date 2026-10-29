# Bloque 1. Sistemas Gestores de Bases de Datos con Docker

## 1. Sistemas de Gestión de Bases de Datos con Docker (SGBD)

Ante la necesidad de disponer de un SGBD para nuestras aplicaciones, podemos tener diversas alternativas. Habitualmente era necesario instalar un servidor en nuestro equipo, lo que a la larga hace que tengamos el sistema saturado y con problemas de rendimiento. Para evitar esto, podemos optar por utilizar **Docker**.

**Docker** creará un contenedor que aloje una base de datos. Optaremos por **MySQL**, aunque también se pueden utilizar otros SGBD como PostgreSQL, MongoDB, etc.

## 2. ¿Qué es Docker?

Docker es una herramienta que puede empaquetar una aplicación y sus dependencias en un contenedor virtual que puede ejecutarse en cualquier servidor Linux. Esto aporta flexibilidad y portabilidad a las aplicaciones, ya que estas pueden ejecutarse en instalaciones físicas, en la nube pública, en nubes privadas, etc.

Las principales ventajas del uso de contenedores son:

- La flexibilidad, ya que incluso las aplicaciones más complejas pueden incluirse en contenedores.
- La baja carga que suponen para el sistema, al compartir el mismo kernel que el anfitrión.
- La posibilidad de desplegar actualizaciones en caliente.
- La portabilidad, ya que pueden desarrollarse localmente, desplegarse en la nube y ejecutarse en cualquier lugar.
- La escalabilidad, ya que permite incrementar automáticamente réplicas de los contenedores.
- Los servicios en contenedores pueden apilarse *on the fly*.

!!! warning
    ### Conceptos clave

    Es importante tener claros los siguientes conceptos:

    - **Una imagen** es un paquete ejecutable que incluye todo lo necesario para ejecutar una aplicación: el código, el entorno de ejecución, las librerías, las variables de entorno y los archivos de configuración. Una vez generada, la imagen no se modifica, de forma equivalente a los antiguos archivos ISO. Puede considerarse como una plantilla para crear contenedores.
    - **Un contenedor**, por su parte, es una instancia en ejecución de una imagen; es decir, lo que se crea cuando ponemos en marcha una imagen. El contenedor sí puede modificarse internamente durante su ejecución.

    Podríamos decir que un contenedor es a una imagen lo que un proceso es a un programa:

    ```bash
    Proceso = Programa en ejecución
    Contenedor = Imagen en ejecución
    ```

En la página [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/) podéis encontrar la guía de instalación de Docker para diferentes sistemas operativos. Además del sistema base Docker, podéis añadir un entorno gráfico como Docker Desktop o Portainer para facilitar la gestión de los contenedores.

![Imagen generada por el autor con Gemini.](img/cap1.png)

Imagen generada por el autor con Gemini.

## 3. ¿Cómo crear un contenedor de MySQL con Docker?

Para crear un contenedor de MySQL con Docker, podéis utilizar el siguiente comando. El comando es una única línea, pero se ha dividido en varias líneas para facilitar su lectura:

```bash
docker run--name mysql-srv \-p3308:3306 \-eMYSQL_ROOT_PASSWORD="toor" \-d mysql:latest
```

Este comando, la primera vez, descargará la imagen de MySQL (`docker pull` de forma automática) y creará el contenedor. PERO NO LO EJECUTÉIS AÚN.

!!! warning "Peligro"
    El comando `docker run` solo debe ejecutarse la primera vez, ya que ejecutar `docker run` crea un contenedor nuevo.

    Si el contenedor ya existe, debe gestionarse mediante:

    - Iniciarlo con `docker start mysql-srv`.
    - Detenerlo con `docker stop mysql-srv`.

!!! question
    ### ¿Dónde están los datos?

    Nuestras bases de datos, tal y como las hemos definido, se guardarán en el sistema de archivos del contenedor. Por tanto, si el contenedor se elimina, también se perderán los datos. Para realizar este curso, esto sería suficiente.

    En un entorno de producción esto no sería aceptable, ya que perderíamos todos los datos cada vez que el contenedor se eliminara o reiniciara. Para evitarlo, podemos utilizar **volúmenes de Docker**, que permiten almacenar los datos fuera del contenedor y mantenerlos persistentes aunque el contenedor se elimine o se reinicie.

    Para ello, bastaría con añadir al comando `docker run` la siguiente línea:

    ```bash
    -v /path/to/local/directory:/var/lib/mysql \
    ```

    Existen dos formas de persistir los datos de un contenedor:

    - **Bind mount**: monta un directorio del sistema operativo (`C:\...` o `/home/...`).
    - **Volumen de Docker**: Docker gestiona automáticamente el almacenamiento de los datos (`-v mysql-data:/var/lib/mysql`).

    Para este curso se recomienda utilizar un **volumen de Docker**, ya que es más sencillo y portable.

!!! question
    ### Comando final

    ```bash
    docker run --name mysql-srv \
    	-p 3308:3306 \
    	-e MYSQL_ROOT_PASSWORD="toor" \
    	-v mysql-srv-data:/var/lib/mysql \
    	-d mysql:latest
    ```

    ### ¿Qué hace cada parámetro?

    - `--name mysql-srv` → Asigna el nombre `mysql-srv` al contenedor.
    - `-p 3308:3306` → Expone el puerto `3308` de tu máquina al puerto `3306` del contenedor.
    - `-e MYSQL_ROOT_PASSWORD="toor"` → Establece la contraseña del usuario `root`.
    - `-v mysql-srv-data:/var/lib/mysql` → Crea un volumen persistente llamado `mysql-srv-data` donde se almacenarán las bases de datos.
    - `-d mysql:latest` → Descarga (si es necesario) y ejecuta la última versión de MySQL en segundo plano.

Con todo esto damos por supuesto que ya tenemos un SGBD MySQL en funcionamiento.

Adicionalmente, se recomienda disponer de una herramienta de gestión de bases de datos como **MySQL Workbench** o **DBeaver** para facilitar la administración de las bases de datos, la ejecución de consultas SQL o, simplemente, revisar qué está ocurriendo en la base de datos.

A continuación veremos cómo podemos conectarnos a esta base de datos desde nuestras aplicaciones Java, primero utilizando el método clásico con **JDBC** y posteriormente mediante un **ORM** como **Hibernate**.

## 4. Comandos útiles de Docker

| Comando | Descripción | Ejemplo |
| --- | --- | --- |
| `docker pull` | Descarga una imagen desde Docker Hub sin ejecutarla. | `docker pull mysql:latest` |
| `docker run` | Crea y ejecuta un nuevo contenedor a partir de una imagen. | `docker run --name mysql-srv -p 3308:3306 -e MYSQL_ROOT_PASSWORD="toor" -v mysql-srv-data:/var/lib/mysql -d mysql:latest` |
| `docker ps` | Muestra los contenedores que están actualmente en ejecución. | `docker ps` |
| `docker ps -a` | Muestra todos los contenedores, incluidos los detenidos. | `docker ps -a` |
| `docker images` | Lista todas las imágenes almacenadas localmente. | `docker images` |
| `docker start` | Inicia un contenedor que estaba detenido. | `docker start mysql-srv` |
| `docker stop` | Detiene un contenedor en ejecución. | `docker stop mysql-srv` |
| `docker restart` | Reinicia un contenedor. | `docker restart mysql-srv` |
| `docker rm` | Elimina un contenedor detenido. | `docker rm mysql-srv` |
| `docker rmi` | Elimina una imagen del sistema. | `docker rmi mysql:latest` |
| `docker logs` | Muestra los registros (logs) generados por un contenedor. | `docker logs mysql-srv` |
| `docker exec` | Ejecuta un comando dentro de un contenedor en ejecución. | `docker exec -it mysql-srv bash` |
| `docker inspect` | Muestra información detallada de un contenedor o imagen en formato JSON. | `docker inspect mysql-srv` |
| `docker volume ls` | Lista todos los volúmenes disponibles. | `docker volume ls` |
