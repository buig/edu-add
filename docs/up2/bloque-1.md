# Bloque 1. Conexión y creación de la base de datos

> Apuntes actualizados a Python 3, con ejemplos orientados a objetos (POO) y buenas prácticas de seguridad.

> 💻 **Material de apoyo**: junto a estos apuntes se entrega un proyecto de ejemplo completo y ejecutable (`schema_employees.sql`, `bbdd.py`, `demo_tema2.py`) que crea esta misma base de datos y recorre, paso a paso, todos los ejemplos del tema contra datos reales. Todo el código de este documento está escrito para encajar exactamente con ese esquema.

## 1. Gestores de bases de datos

Un **sistema gestor de bases de datos** (SGBD) es el conjunto de programas que administra y gestiona la información contenida en una base de datos, proporcionando distintos niveles de abstracción según el tipo de usuario: normalmente ni el programador ni el usuario final necesitan saber cómo ni dónde se almacenan físicamente los datos.

Podemos distinguir dos tipos de SGBD según su relación con la aplicación:

- **SGBD embebido**: no se ejecuta como un proceso independiente, sino que forma parte de la propia aplicación. Ejemplo típico: **SQLite**.
- **SGBD independiente**: se ejecuta como un proceso propio, separado de la aplicación que lo usa, y se administra de forma independiente. Ejemplos: **MySQL**, **PostgreSQL**, **MariaDB**, **SQL Server**.

| | Embebido (SQLite) | Independiente (MySQL/PostgreSQL) |
|---|---|---|
| Proceso | Parte de la aplicación | Proceso propio del sistema operativo |
| Acceso concurrente | Limitado | Múltiples clientes a la vez |
| Instalación | Ninguna (un fichero) | Requiere instalar y configurar un servidor |
| Uso típico | Apps de escritorio/móvil, prototipos | Aplicaciones web, sistemas multiusuario |

---

## 2. Modelo relacional vs. modelo orientado a objetos

### Modelo relacional

Organiza los datos en **tablas**: las **columnas** representan un mismo concepto (atributo) y las **filas** (tuplas o registros) representan entidades individuales. Para representar información compleja se usan **varias tablas relacionadas entre sí mediante claves foráneas**.

Es un modelo sencillo, maduro y muy eficiente, que domina el mercado desde hace décadas gracias a su solidez. Tiene limitaciones a la hora de representar información poco estructurada o con relaciones muy dinámicas y complejas — ahí es donde entran en juego las bases de datos NoSQL (documentales, clave-valor, grafos...), que complementan al modelo relacional sin sustituirlo.

### Modelo orientado a objetos

En el modelo orientado a objetos (el que usamos al programar en Python), los **objetos** representan entidades, características, procesos o relaciones del dominio, combinando **datos** (atributos) y **comportamiento** (métodos).

Cuando programamos aplicaciones que usan bases de datos relacionales, trabajamos constantemente en la frontera entre ambos mundos: los datos viven en tablas, pero los manipulamos como objetos en nuestro código Python. A ese "salto" entre ambos modelos se le llama **impedancia objeto-relacional**, y es el problema que resuelven herramientas como los **ORM** (*Object-Relational Mapper*, p. ej. SQLAlchemy o el ORM de Django), que veremos más adelante en el ciclo. En este tema trabajaremos con el modelo relacional "a mano", usando el conector oficial de MySQL, para entender bien qué hay por debajo de un ORM.

---

## 3. Connector/Python

**Connector/Python** (`mysql.connector`) es la interfaz que Python ofrece para conectarse a un SGBD relacional. Expone una **API común**: sea cual sea el motor de base de datos, el código para conectar, consultar, insertar o cerrar la conexión es muy similar. Lo único que cambia realmente entre motores es el **driver** concreto y los datos de la cadena de conexión.

Esto tiene una consecuencia práctica muy importante: si programamos bien nuestra aplicación (separando la lógica de acceso a datos del resto del código, como haremos con el patrón *repositorio*), **cambiar de motor de base de datos en el futuro —o soportar varios— cuesta muy poco**.

| SGBD | Paquete / driver habitual |
|---|---|
| MySQL / MariaDB | `mysql-connector-python` (oficial de Oracle) |
| PostgreSQL | `psycopg2` / `psycopg` |
| SQLite | `sqlite3` (incluido en la librería estándar) |
| SQL Server | `pyodbc` |

En este tema usaremos **MySQL** con `mysql-connector-python`, pero la forma de trabajar (conectar, cursor, ejecutar, commit, cerrar) es la misma con cualquier otro.

---

## 4. Instalación y configuración

```bash
pip install mysql-connector-python
```

> ⚠️ **Advertencia — credenciales**: nunca escribáis el usuario y la contraseña de la base de datos directamente en el código fuente, y mucho menos lo subáis a un repositorio Git. Usad variables de entorno o un fichero `.env` (con el paquete `python-dotenv`) que **no** se suba al control de versiones (añadidlo a `.gitignore`).

```bash
pip install python-dotenv
```

```python
# .env  (no se sube al repositorio)
DB_HOST=127.0.0.1
DB_USER=scott
DB_PASSWORD=tiger
DB_NAME=employees
```

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    "host": os.getenv("DB_HOST"),
    "user": os.getenv("DB_USER"),
    "password": os.getenv("DB_PASSWORD"),
    "database": os.getenv("DB_NAME"),
}
```

A partir de aquí, cualquier clase que necesite conectarse recibirá `DB_CONFIG` como parámetro, en lugar de tener las credenciales escritas dentro.

---

## 5. Conexión a la base de datos (POO)

Encapsulamos la conexión en una clase que además funciona como **gestor de contexto**, para poder usar `with` y que la conexión se cierre siempre, incluso si ocurre un error:

```python
import mysql.connector
from mysql.connector import errorcode

class ConexionBD:
    """Gestor de contexto para una conexión MySQL.

    Uso:
        with ConexionBD(DB_CONFIG) as cnx:
            cursor = cnx.cursor()
            ...
    """

    def __init__(self, config: dict):
        self.config = config
        self.cnx = None

    def __enter__(self) -> mysql.connector.MySQLConnection:
        try:
            self.cnx = mysql.connector.connect(**self.config)
        except mysql.connector.Error as err:
            if err.errno == errorcode.ER_ACCESS_DENIED_ERROR:
                raise RuntimeError("Usuario o contraseña incorrectos") from err
            elif err.errno == errorcode.ER_BAD_DB_ERROR:
                raise RuntimeError(f"La base de datos '{self.config.get('database')}' no existe") from err
            raise
        return self.cnx

    def __exit__(self, exc_type, exc_value, traceback) -> None:
        if self.cnx is not None and self.cnx.is_connected():
            self.cnx.close()

# --- Uso ---
with ConexionBD(DB_CONFIG) as cnx:
    print(cnx.is_connected())   # True
# al salir del bloque with, la conexión ya está cerrada
```

> 💡 **Nota**: desde versiones recientes de `mysql-connector-python`, tanto la conexión como el cursor ya implementan por sí mismos el protocolo de gestor de contexto (`with mysql.connector.connect(**config) as cnx:` y `with cnx.cursor() as cursor:` funcionan directamente). Aun así, mantener nuestra propia clase `ConexionBD` es útil para centralizar la gestión de errores de conexión en un solo sitio.

---

## 6. Creación de la base de datos y las tablas

Seguimos usando como ejemplo la típica base de datos de empleados (`employees`, `departments`, `salaries`...). En vez de un script suelto, lo encapsulamos en una clase responsable únicamente de crear el esquema:

```python
class EsquemaEmpleados:
    """Responsable de crear la base de datos y las tablas si no existen."""

    NOMBRE_BD = "employees"

    TABLAS = {
        "employees": (
            "CREATE TABLE employees ("
            "  emp_no INT NOT NULL AUTO_INCREMENT,"
            "  birth_date DATE NOT NULL,"
            "  first_name VARCHAR(14) NOT NULL,"
            "  last_name VARCHAR(16) NOT NULL,"
            "  gender ENUM('M','F') NOT NULL,"
            "  hire_date DATE NOT NULL,"
            "  PRIMARY KEY (emp_no)"
            ") ENGINE=InnoDB"
        ),
        "departments": (
            "CREATE TABLE departments ("
            "  dept_no CHAR(4) NOT NULL,"
            "  dept_name VARCHAR(40) NOT NULL,"
            "  PRIMARY KEY (dept_no),"
            "  UNIQUE KEY (dept_name)"
            ") ENGINE=InnoDB"
        ),
        "dept_emp": (
            "CREATE TABLE dept_emp ("
            "  emp_no INT NOT NULL,"
            "  dept_no CHAR(4) NOT NULL,"
            "  from_date DATE NOT NULL,"
            "  to_date DATE NOT NULL,"
            "  PRIMARY KEY (emp_no, dept_no, from_date),"
            "  CONSTRAINT fk_dept_emp_emp FOREIGN KEY (emp_no)"
            "    REFERENCES employees (emp_no) ON DELETE CASCADE,"
            "  CONSTRAINT fk_dept_emp_dept FOREIGN KEY (dept_no)"
            "    REFERENCES departments (dept_no) ON DELETE CASCADE"
            ") ENGINE=InnoDB"
        ),
        "salaries": (
            "CREATE TABLE salaries ("
            "  emp_no INT NOT NULL,"
            "  salary INT NOT NULL,"
            "  from_date DATE NOT NULL,"
            "  to_date DATE NOT NULL,"
            "  PRIMARY KEY (emp_no, from_date),"
            "  CONSTRAINT fk_salaries_emp FOREIGN KEY (emp_no)"
            "    REFERENCES employees (emp_no) ON DELETE CASCADE"
            ") ENGINE=InnoDB"
        ),
    }

    def __init__(self, config_sin_bd: dict):
        """config_sin_bd: configuración de conexión SIN la clave 'database',
        porque la base de datos puede que todavía no exista."""
        self.config = config_sin_bd

    def crear_si_no_existe(self) -> None:
        with ConexionBD(self.config) as cnx:
            cursor = cnx.cursor()
            self._crear_base_de_datos(cursor)
            cursor.execute(f"USE {self.NOMBRE_BD}")
            self._crear_tablas(cursor)
            cursor.close()

    def _crear_base_de_datos(self, cursor) -> None:
        try:
            cursor.execute(
                f"CREATE DATABASE IF NOT EXISTS {self.NOMBRE_BD} "
                "DEFAULT CHARACTER SET 'utf8mb4'"
            )
        except mysql.connector.Error as err:
            raise RuntimeError(f"No se pudo crear la base de datos: {err}") from err

    def _crear_tablas(self, cursor) -> None:
        for nombre, sentencia_create in self.TABLAS.items():
            try:
                print(f"Creando tabla {nombre}... ", end="")
                cursor.execute(sentencia_create)
                print("OK")
            except mysql.connector.Error as err:
                if err.errno == errorcode.ER_TABLE_EXISTS_ERROR:
                    print("ya existe")
                else:
                    raise

# --- Uso ---
config_sin_bd = {k: v for k, v in DB_CONFIG.items() if k != "database"}
EsquemaEmpleados(config_sin_bd).crear_si_no_existe()
```

> Nótese el uso de `CREATE DATABASE IF NOT EXISTS` y la comprobación del código de error `ER_TABLE_EXISTS_ERROR`: evitamos que el programa falle simplemente porque ya hayamos ejecutado el script antes. Como se indica en la documentación de MySQL, crear y eliminar tablas es una operación costosa — normalmente nos conectaremos a tablas que ya existen, y solo las crearemos la primera vez (o en tablas temporales).

---
